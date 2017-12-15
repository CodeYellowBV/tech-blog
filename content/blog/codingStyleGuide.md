+++
title = "Coding style guide"
slug = "coding-style-guide"
date = "2013-07-05"
description = ""
aliases = ["/coding-style-guide.html"]
keywords = ["PHP", "Javascript"]
categories = [""]
tags = ["PHP", "Javascript"]
+++

author: Burhan Zainuddin


<img alt="test" title="marionette" src="/images/codingStyleGuide.jpg" style="
    padding: 5px;
    margin: 0px auto;
    display: block;
    height: 150px;
" />

# Introduction

The 2 most used languages at Code Yellow are

1. PHP
2. Javascript

PHP has [PHP-FIG](https://github.com/php-fig/fig-standards) which is becoming more commenly accepted. Javascript has [idiomatic](https://github.com/rwldrn/idiomatic.js). For all projects we apply these rules:

# General

- First check:
    1. [PSR-1](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md)
    2. [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md)

- Use single quotes for strings where possible:
    - PHP
        $stringA = 'this is a string';
        $stringB = 'this is another string ' . $blah . ' test';
    - Javascript
        var stringA = 'this is a string',
        stringB = 'this is a string ' + blah + ' test';

- First start with properties, then functions.

- Prefix variables with class type where possible. Uppercase letters are classes, lowercase objects. The available prefixes are
    - M, m for model
    - V, v for view
    - T for template
    - X for mixin
```php
    <?php
    // PHP
    $mPerson = new \Model_Person();

    class SomeSpecialClass {
        private $MPerson = '\\Model_Person';
        private $mPerson = null;

        public function __construct(\Model_Person $mPerson = null) 
        {
            if ($mPerson === null) {
                $this->mPerson = $mPerson;
            } else {
                $this->mPerson = new $this->MPerson();
            }
        }

        // Strict camelCase.
        public function parseRestHttp() 
        {
            // Do something.
        }

    }
```
```javascript
    // Javascript
    var MPerson = require('model/person'),
    mPerson = new MPerson();
```

# Javascript

[Idiomatic](https://github.com/rwaldron/idiomatic.js/) with adjustments. Check out [backbone-crux](https://bitbucket.org/codeyellow/backbone-crux) for an example on how to implement the style guide in Javascript.

- Whitespace
    - Never mix spaces and tabs.
    - 4 spaces for indents

- Parens, Braces, Linebreaks

```javascript
        if (condition) {
            doSomething();
        }

        while (condition) {
            iterating++;
        }

        for (i = 0; i < 100; i++) {
            someIterativeFn();
        }
```

- Using only one `var` per scope (function) promotes readability and keeps your declaration list free of clutter (also saves a few keystrokes).
```javascript
        var foo = 'bar',
        bar = '',
        array = [],
        object = {};
        quux;
```
- var statements should always be in the beginning of their respective scope (function). Same goes for const and let from ECMAScript 6.
```javascript
        function foo(arg1, argN) {
            var bar = '',
            qux;

            // All statements after the variables declarations.
            require('someModule');
        }
```
- Usage.
```javascript
        foo(arg1, argN);
```
- Named Function Declaration.
```javascript
        function square(number) {
            return number * number;
        }
```
- Usage.
```javascript
        square(10);
```

- Really contrived continuation passing style.
```javascript
        function square(number, callback) {
            callback(number * number);
        }

        square(10, function (square) {
            // callback statements
        });
```

- Function Expression
```javascript
        var square = function (number) {
            // Return something valuable and relevant
            return number * number;
        };
```
- Function Expression with Identifier. This preferred form has the added value of being able to call itself and have an identity in stack traces.
```javascript
        var factorial = function factorial (number) {
            if (number < 2) {
                return 1;
            }

            return number * factorial(number - 1);
        };
```

- Constructor Declaration
```javascript
        function FooBar(options) {
            this.options = options;
        }
```
- Usage.
```javascript
        var fooBar = new FooBar({a: 'alpha'});

        fooBar.options;
```


- Functions with callbacks.
```javascript
        foo(function () {
        });

        foo(['alpha']);
        foo(['alpha', 'beta']);
        foo([
            'alpha', 
            'beta', 
            'gamma'
        ]);

        foo({a: 'alpha'});

        foo({
            a: 'alpha',
            b: 'beta'
        });

        foo('bar');
```

- Where possible, be concise.
```javascript
    // Instead of...
    if (array.length > 0) ...
    if (array.length === 0) ...
    if (string !== '') ...
    if (string === '') ...
    if (foo === true) ...
    if (foo === false) ...
    
    // ...use
    if (array.length) ...
    if (!array.length) ...
    if (string) ...
    if (!string) ...
    if (foo) ...
    if (!foo) ...

    // Be careful, this will also match: 0, '', null, undefined, NaN
    // If you _MUST_ test for a boolean false, then use
    if (foo === false) ...

    // When only evaluating a ref that might be null or undefined, but NOT false, "" or 0,
    // instead of this:
    if (foo === null || foo === undefined) ...

    // ...take advantage of == type coercion, like this:
    if (foo == null) ...

    // Remember, using == will match a `null` to BOTH `null` and `undefined`
    // but not `false`, "" or 0
    null == undefined
```
- ALWAYS evaluate for the best, most accurate result - the above is a guideline, not a dogma. Type coercion and evaluation notes. Prefer `===` over `==` (unless the case requires loose type evaluation) === does not coerce type, which means that:
```javascript
    '1' === 1; // false
```    
- == does coerce type, which means that:
```javascript
    '1' == 1; // true
```

- 'Namespace' with colon:
```javascript
    vent.trigger('nav:open');
    vent.trigger('nav:close');
```

- Add wrappers before subject:
```javascript
    vent.trigger('before:nav:close');
    vent.trigger('after:nav:close');
```

- Camelcase:
```javascript
    'slideToggle'
    'before:slideToggle'
    'after:slideToggle'
```
- Prefixe models, collections, views, regions and mixins as follows:
```javascript
    // Class:
    var MAccount = require('model/account'),
    CAccount = require('collection/account'),
    VAccount = require('view/account'),
    RDialog = require('region/dialog'),
    XAccount = require('mixin/account'),

    // Instance of class:
    mAccount = new MAccount(),
    cAccount = new CAccount(),
    vAccount = new VAccount(),
    xAccount = new XAccount()
```
- File header:

```
    // # Some awesome code title
    //
    // ### _Subtitle._
    //
    // Amazing description of code.
    // ___
    //
    // **Author:** AB Zainuddin
    //
    // **Email:** burhan@codeyellow.nl
    //
    // **Website:** 
    //
    // **License:** Not yet...
    // ___
```
- Document functions with PHPDoc/JSDoc.

- Always 'use strict';

- Comment above lines.

- Use these settings for .jshintrc:
```javascript
    {
        "globals": {
            // RequireJS
            "require": true,
            "define": true,
            "module": true,

            // Jasmin.
            "using": true,
            "expect": true,
            "it": true,
            "describe": true,
            "waitsFor": true,
            "runs": true,
            "jasmine": true,
            "beforeEach": true,
            "mostRecentAjaxRequest": true
        },
        "onecase": true,
        "quotmark": "single",
        "unused": "vars",
        "strict": true,
        "undef": true,
        "trailing": true,
        "browser": true
    }

```
# Frontend project layout

- Use camelCase for file names.

- Put template in same folder as Javascript file
    - search.html
    - search.js

- If a file uses other files, put it in a same named folder. Example:
    - search.html
    - search.js
    - search/
        - result/
            - item.html
            - item.js
        - pager.html
        - pager.js
        - result.html
        - result.js

- Use _ prefix for system folders.
    - view/
        - _common/
            - search.js
        - user/
            - overview.js (<-- uses _common/search.js)

- Route / convert to : events.
    - user/overview
    - user:overview

    - user/create
    - user:create

    - user/edit/:id
    - user:edit

- Routes map to files in view folder where possible.
    - user/overview
    - user/overview.js

    - user/create
    - user/create.js

    - user/edit
    - user/edit.js



# HTML

- Prefix elemenets required by app with with an underscore.
- Use double quotes for attribute values.
- Suffixed region DOM-element selectors with '-region'.
- Classname seperatered with '-'.
- Avoid using ids.

```html
<div id="_body"></div>
<div class="_some-important-element"></div>
<div class="_product-region"></div>
```
