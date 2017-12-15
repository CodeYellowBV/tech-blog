+++
title = "Refactoring our frontend development stack"
slug = "refactoring-our-frontend-development-stack"
date = "2015-10-22"
description = ""
aliases = ["/refactoring-our-frontend-development-stack.html"]
keywords = ["Software Development", "Javascript"]
categories = [""]
tags = ["Software Development", "Javascript"]
+++

author: Kees Kluskens

At Code Yellow we write frontend heavy apps. This means that the browser is responsible for all the rendering, and the backend only handles API requests.

In the last few weeks we have been busy refactoring a large part of our backend, frontend and development workflow to ensure the best technologies available are used.

In this article I want to highlight the changes we made to our frontend stack. As such, this article is intended for frontend developers.

## Ditching Gulp
We were previously using [Gulp](http://gulpjs.com/) to help build the frontend. When we saw more and more packages ditching Gulp for npm scripts, we started experimenting with this.

Letʼs take a look at how we lint Javascript files with Gulp.

To lint Javascript files with Gulp, using [ESLint](http://eslint.org/), and let it exit with exit code `1` when errors are found, you have to do something like this:

```js
var eslint = require('gulp-eslint');

gulp.task('lint', function() {
    return gulp.src(config.files)
        .pipe(eslint())
        .pipe(eslint.format())
        .pipe(eslint.failAfterError());
});
```

With npm scripts you could do this with less lines of code _and_ ditch the `gulp-eslint` dependency. It uses the CLI implementation that comes with the ESLint package:

```json
{
    "scripts": {
        "lint": "eslint ."
    }
}
```

The `gulp-eslint` dependency depends on `eslint`. After removing this, we can now include `eslint` directly. Now we can control the version of this dependency directly :).

Next we want to ditch Gulp for building the frontend. Javascript is built with RequireJS (by using `spawn` in the Gulpfile), CSS with Sass using `gulp-sass`.

## Module bundling
For Javascript bundling, we had been using [RequireJS](http://requirejs.org/) for a few years. It worked, but development is stagnant and we started to see interesting development on [Browserify](http://browserify.org/) and [webpack](https://webpack.github.io/). webpack seemed very interesting because of its philosophy: all assets are just modules. Why treat CSS, an image or a font any different then a Javascript file? Itʼs just as much part of the frontend as Javascript is.

Pete Hunt did a good job convincing us of this with his talk about “[How Instagram.com Works](https://www.youtube.com/watch?v=VkTCL6Nqm6Y)”.

By treating all our assets as modules, we gain the following advantages;

- Caching: webpack can process the module and append a `md5` hash of the file to the filename (necessary for proper cache busting). This works in all modules, so even CSS!
- Compile time errors: If a font or image is deleted, renamed or whatever, and is still referenced, the build will fail instead of silently succeeding.
- Include what you use: Only assets that are actually used are included in the build; no cruft.
- Standalone package: After the build, we only need the resulting output in `dist/`; everything else can be removed.

The biggest disadvantage of webpack is the learning curve, combined with a lacking documentation. By looking at [examples](http://slidedeck.io/unindented/webpack-presentation) [from](http://www.slideshare.net/ssuser0e4922/webpack-slides-51907869) others we could, carefully, forge our own config file.

I will now tell you a bit about what webpack is, and how we use it.

### Loaders

[Loaders](https://webpack.github.io/docs/loaders.html) in webpack are a sort of preprocessors for modules. You can specify a file extension (e.g. `.scss`) and add the loaders you want for that file.

With `.scss` (Sass) files, we want it to pass through [node-sass](https://github.com/sass/node-sass), then [Postcss](https://github.com/postcss/postcss) (for automatic CSS prefixes) and finally through a [CSS loader](https://github.com/webpack/css-loader). This last loader might be a bit confusing, but it just converts the CSS to a Javascript module, so that you can `require` it.

We use [Backbone](http://backbonejs.org/) and [Marionette](http://marionettejs.com/) as frameworks. The template for a view is parsed with the `_.template` function from Lodash.

The boilerplate code for a typical view looked like this:

```js
define(function (require) {
    'use strict';

    var TItem = require('text!./item.html');
    return Marionette.ItemView.extend({
        template: _.template(TItem),
    });
});
```

We started with just using [html-loader](https://github.com/webpack/html-loader) to convert the `.html` file to a module, but soon found out the existence of [underscore-template-loader](https://github.com/emaphp/underscore-template-loader). This loader compiles given file with the underscore / lodash `_.template` function. As a result, the browser doesnʼt need to compile the template at runtime anymore :).

Anyway, the boilerplate code for a view is now dramatically more readable:

```js
import TItem from './item.html';

export default Marionette.ItemView.extend({
    template: TItem,
});
```


### Decoupling frontend and backend

Previously the backend also served a template to start the application. This was necessary because in production we append the version number to the JS / CSS files (cache busting). The template also contains some bootstrapping of the app; passing the current user data, csrf token, app version and environment to Javascript.

Using the backend to serve a template always felt like a hack. The backend should not serve frontend files. Something something _separation of concerns_?

By using `html-webpack-plugin` we can fix the cache busting problem. It knows the filenames of our JS / CSS files, so it can just insert these files in the html. It produces a static html file in `dist/`.

And, a small win, after a change to this file it gets reloaded automatically by webpack.

The rest of the template bootstrapping can also be done using a separate request to the API; with a GET `api/bootstrapper`. This returns the current user data, csrf token etc.

Finally, nginx can be configured to serve a static file at `/`. The backend sits behind `/api/`.

Basically the frontend is just a consumer of the backend now. The backend has no ties to the frontend anymore. For some of our applications we have developed a Chrome extension that also uses the backend. The main frontend is now an equal consumer compared to the Chrome extension.

### Development workflow

[webpack-dev-server](https://webpack.github.io/docs/webpack-dev-server.html) provides us with a simple server and live reloading of all modules. Because CSS and images are modules too, changes in these files also reload the page.

To start the development server, simply type `npm start`. It will read environment variables from a `.env` file (which you can copy from a `.env.example` file). In this file, you can configure a port and host for the development server to your liking.

The most awesome part: it requires _one_ line of configuration.

```json
{
    "scripts": {
        "start": ". .env; webpack-dev-server --output-public-path / --watch-poll --port $CY_WEBPACK_PORT --host $CY_WEBPACK_HOST"
    }
}
```

Developer happiness went way up after we started using webpack-dev-server. It informs you of build errors in the console, even showing a nice arrow pointing to your ludicrous typo. When I first saw this, I cried tears of joy.

## Testing

We had been using Jasmine for some time, together with a package we made that allows you to run [Chromium via the CLI](https://github.com/CodeYellowBV/run-headless-chromium). Chromium runs the test suite and shows you the results in the CLI. This way we can run our tests with a real DOM.

It worked fine, but it has a big disadvantage: you have to install Chromium and [Xvbf](https://en.wikipedia.org/wiki/Xvfb) globally on the server you want the tests to run on, and keep it up-to-date. That means having global dependencies you canʼt manage easily.

So we started looking at alternatives. PhantomJS is extremely popular, but uses an ancient WebKit version created in 2011. We at Code Yellow firmly believe that was the time dinosaurs went extinct.

Another, less popular, alternative is [Selenium WebDriver](https://github.com/SeleniumHQ/selenium). But as soon as we noticed it was using Java, we were appalled. No way this monstrous Oracle-backed closed-source technology is running on our nice little Debian servers!

![Shocked beyond reason.](/images/wuuuut.gif)

Somewhat disappointed from the quick search we started looking at how some packages handle DOM-testing.
Apparently Marionette is using [JsDom](https://github.com/tmpvar/jsdom) for their tests. Simply put, JsDom is a Node.js implementation of the DOM. You can just add it as a npm dependency.

We made a [simple npm package](https://github.com/CodeYellowBV/jsdom-test) that allows you to run it with the CLI and exit with status code `1` if the tests fail. Feed it a html page that contains the test suite and it will run the tests. And mother of god, it is fast!

Weʼre still in an early stage of playing with JsDom, but so far it looks good. The build of webpack also generates a `dist/test.html` file that contains the test suite. Our package, jsdom-test, simply runs the suite. In the `package.json` it looks like this:

```json
{
    "scripts": {
        "build-js": "webpack --colors --progress --bail",
        "clean": "rm -rf dist/*",
        "lint": "eslint .",
        "pretest": "npm run -s build-js",
        "test": "npm run -s lint && jsdom-test dist/test.html"
    }
}
```

Everywhere we saw packages using Mocha as their test runner. Because it seemed more standard, we switched to using Mocha. Itʼs not much better, so existing project will still be using Jasmine.

## Separate as much as possible in packages!

We separate problems that occur often, like pagination, in a package for easy reuse in other packages. As a result, bugs donʼt have to be fixed twice. As long as the code is not specific for your project, why not put it in a package?

You donʼt need to spend much time writing a perfect package. If it works for the project and has no specific code for that project in it, itʼs worth something.

## Open-sourcing our frontend stack

You can find our frontend development stack at [Github](https://github.com/CodeYellowBV/devstack-frontend). We made it open-source mainly to serve as an example. It took a lot of time to configure the whole stack, so we hope it can save time for people. Feel free to copy the bits you like!
