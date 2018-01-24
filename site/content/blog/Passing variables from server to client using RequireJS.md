+++
title = "Passing variables from server to client using RequireJS"
slug = "passing-variables-from-server-to-client-using-requirejs"
author = "Burhan Zainuddin"
date = "2013-12-10"
description = ""
aliases = ["/passing-variables-from-server-to-client-using-requirejs.html"]
keywords = ["RequireJS", "Javascript"]
tags = ["RequireJS", "Javascript"]
+++

It's quite common to pass variables from server to client. Common uses are bootstrapping data, syncing config setting etc.
Consider the following scenario: a single page app where a user logs in and refreshes the current page. You want the user to
still be logged in. The most commonly used practise is to put it in a script tag:

```html
<script type="text/javascript">
    var userId = 1;
</script>
```

This way you introduce a global variable userId. What happens if the number of variables increase? Probably "namespace" it:

```html
<script type="text/javascript">
    var config = {
        userId: 1,
        bootstrap: {
            product: {
                id: 1,
                title: 'Awesome product',
            }
        }
    };
</script>
```

Now there is only 1 global variable. And maybe AMD it in config.js: 

```javascript
define(function (require) {
    return window.config ? window.config : {};
});
```

## Module config

RequireJS supports [module configs](http://requirejs.org/docs/api.html#config-moduleconfig). 
Lets try rewriting using module config:

## 1. Server generated HTML:
```html
<!-- First define config settings. -->
<script type="text/javascript">
    var require = {
        config: {
            // Config options for a module named 'config'.
            config: {
                userId: 1
            }
        }
    };
</script>

<!-- Load RequireJS after defining config settings. -->
<script type="text/javascript" src="require.js" data-main="app"></script>
```

## 2. Create a new module named 'config':

```javascript
define(function (require) {
    'use strict';

    var module = require('module');

    return module.config ? module.config() : {};
});
```

## 3. And require it somewhere in your app:

```javascript
var config = require('config');
```
