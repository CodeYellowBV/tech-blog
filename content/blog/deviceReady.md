+++
title = "Rapid Cordova application development"
slug = "rapid-cordova-application-development"
date = "2014-02-05"
description = ""
aliases = ["/rapid-cordova-application-development.html"]
keywords = ["Software Development", "Open Source", "Cordova"]
categories = [""]
tags = ["Software Development", "Open Source", "Cordova"]
+++

author: Peter Bex

Developing [Apache Cordova](http://www.cordova.io/) (aka
[Phonegap](http://www.phonegap.com)) applications tends to be rather
painful: the long waits while compiling an application for Android and
uploading it to the device (or emulator) leads to unacceptable round
trip times, especially for applications with lots of large content
files (images, videos).  In this blog post we'll explain how we
develop Cordova applications directly in the desktop web browser, and
announce an open source release of a RequireJS module that helps
making this a little easier for us.

The main reason we've chosen Cordova over native development is
*portability*: writing a native app for each platform you want to
support is a waste of developer effort and, by extension, money.  We
would also like to leverage this portability to the fullest by
developing applications from the browser, as much as we can.  In a
browser, pressing "refresh", results in an **instantaneous** update.

Waiting for the (non-existent) device
=====================================

Very basic applications can be developed in the browser without
problems, but this isn't so easy once you start using some Cordova
plugins to make use of the device's more advanced capabilities.  When
you use these, you generally need to wait for the
[`deviceready`](http://docs.phonegap.com/en/2.0.0/cordova_events_events.md.html#deviceready)
event to get triggered.  This signals that Cordova has finished
setting up all plugins and handlers and you can start using these.
The initial "demo" application which gets installed when you
initialise a new Cordova project also uses the `deviceready` event as
a sort of starting point for the application.  This means that when
you open it in your web browser, it won't seem to "do" anything,
because it's waiting for an event which will never arrive.  This is
what you'll see when you open it in a browser:

![connecting to device...](/images/phonegap-waiting-for-deviceready.png)

Because the event is never triggered, it will stay stuck in this state
rather than giving an error.  On an actual device, this is what you'd
see:

![device is ready](/images/phonegap-after-deviceready.png)

What we really want is for the application to detect that it's not
running on a device and continue the application with that knowledge.
Furthermore, we are using the excellent
[RequireJS](http://requirejs.org/) package as a module system.  To
integrate cleanly into the rest of our code base, we would like to
specify that our code depends on device capabilities in a declarative
fashion, much like the [RequireJS DOM Ready
plugin](http://requirejs.org/docs/api.html#pageload) allows us to
specify "the DOM being ready" as a "dependency" of our module.  It
turns out that this isn't very difficult.

Detecting the Cordova environment
=================================

Phonegap/Cordova is set up in such a way that the initial HTML page
contains a `<script>` tag which loads a non-existent `phonegap.js` or
`cordova.js` file.  The reason for this is that each supported
platform (Android, iOS, BlackBerry, ...) has its own
JavaScript-to-native "bridge" which allows JavaScript to invoke native
code.  So the `phonegap.js` file is installed in the platform-specific
directory when you add a new platform.  Then, when you "build" your
application, your own files are installed alongside these
previously-installed files in the platform directory, and the whole
"environment" is bundled up and uploaded to the device, where the
`<script>` tag works as expected.

Because this phonegap-script is not loaded on the desktop, none of the
variables it declares are available.  We can make use of this to
detect the presence of Cordova, simply by checking whether
`window.cordova` is defined.  This allows to register the
`deviceready`-handler only when we're actually on the device, and
calling the same handler with a different value on the desktop.  The
callback handler can then dispatch to load different code depending on
the situation.

We have wrapped up this logic into a nice RequireJS component, which
we are releasing today as free software.  It is dual licensed under
the "new BSD" and MIT licenses.  Check it out on
[GitHub](https://github.com/CodeYellowBV/deviceReady)!

Additionally, we've published it as [a Bower
component](http://sindresorhus.com/bower-components/#!/search/requirejs-deviceready),
to make integrating it into your code base even easier.


Case study: database support
============================

As it turns out, the native "web view" components in iOS and Android
(which Cordova uses internally) make absolutely no guarantees about
the persistence and maximum available space in **any** of the "HTML5"
storage options.  For some versions of Android, the amount of data you
can store in a Web SQL database and HTML Local Storage is [as low as 5
MB](https://groups.google.com/forum/#!searchin/phonegap/storage$20limit/phonegap/dOZjjEgwF_U/jQtG997bDN4J).
iOS is even worse: in newer versions it may ["randomly" decide to drop
your "persistent" data](https://issues.apache.org/jira/browse/CB-330).

This presents us with a big limitation: we want our data to stay safe,
and we can't know in advance how much storage we'll need.  Failing at
seemingly-arbitrary points in time is not an option!  Luckily, there
is a relatively simple solution: we can use Phonegap plugins to access
native code which **can** make those guarantees.  After some research,
we decided that the [Cordova SQLite
plugin](https://github.com/lite4cordova/Cordova-SQLitePlugin) is our
best option.  It works on iOS, Android and there's also a Windows
mobile version, and it tries to offer an API which conforms to the W3C
[Web SQL Database](http://www.w3.org/TR/webdatabase/) standard.

Even though Web SQL is deprecated and will likely be removed from
browsers in the future, as things stand today it's our best option for
developing in the browser and deploying to a device without making any
changes to the application.  Web SQL does not make any guarantees
about storage space either, but we don't care about that so much when
developing in the browser.

The SQLite plugin doesn't work as a "polyfill" but has its own
interface endpoint: `window.sqlitePlugin`.  Furthermore, it will only
be available from JavaScript after the `deviceready` event has fired.
The code to make this difference "invisible" is very simple using our
RequireJS plugin.  Our application's `sqldb` module looks like this:

```js
define(['deviceReady!'], function(isCordova) {
  'use strict';
  
  var dbRootObject = isCordova ? window.sqlitePlugin : window;
  
  if (typeof dbRootObject.openDatabase == 'undefined') {
    window.alert('Your browser has no SQL support!  Please try a Webkit-based browser');
    return null;
  }
  
  var db = dbRootObject.openDatabase('my-database', '', 'My database', null),
  // ...Some more convenience and specific schema creation code...
  return db;
});
```

With this, the underlying database implementation is completely
transparent to the code that uses it, which can simply `require` the
above module:

```js
var sqldb = require('sqldb');
sqldb.transaction(function(t) {
  t.executeSql('SELECT col1 FROM table WHERE col2 = ?', [value], ...);
});
```

Like I said, support for Web SQL is unfortunately being phased out,
and Mozilla has already taken the step of removing Web SQL from
Firefox.  There's [a WebSQL
addon](https://addons.mozilla.org/en-US/firefox/addon/html5-websql-for-firefox/)
which tries to restore Web SQL support, but it is too basic for our
needs.  Our application will run in production for years to come
(hopefully!), so being able to perform schema migrations is pretty
essential for us. [Who cares about versions,
indeed!](https://addons.mozilla.org/en-US/firefox/files/browse/130768/file/components/websql.js#L139)

Anyway, with the code above we are able to run our application
straight from the file system in Chromium.  There's only one more
hurdle to overcome, and that's this message:

> XMLHttpRequest cannot load file:///home/peter/src/test/www/app/view/myview.html.
> Cross origin requests are only supported for HTTP

If we run the application from `http://localhost` we won't see this,
but because I really like to keep things as simple as possible I would
prefer not to use a web server if I don't strictly have to.  So, I
investigated why we get this message.  It turns out that the
[RequireJS text plugin](https://github.com/requirejs/text) uses
XMLHttpRequest to load resources, whereas regular RequireJS injects
`<script>` tags into the HTML.  The documentation for the text
plugin states:

> Many browsers do not allow file:// access to just any file. You are better off serving the application from a local web server than using local file:// URLs. You will likely run into trouble otherwise.

Never one to take other people's well-meant advice, I figured out that
Firefox won't complain like this (probably assuming that everything
loaded over the file:// protocol is from the same "domain"), and in
Chromium you can simply pass a flag when starting it:

```sh
	$ chromium --allow-file-access-from-files
```

And that's all there's to it!
