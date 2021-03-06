**Important notice:** Tarp.require is the replacement of [Smoothie](https://github.com/letorbi/tarp.require/tree/smoothie).
It introduces a number of new features and improvments, so it is recommended to use Tarp.require from now on. Please
read [the migration documentation](https://github.com/letorbi/tarp.require/blob/master/doc/migration.md) for further
information.

//\ Tarp.require - a lightweight JavaScript module loader
=========================================================
Tarp.require is a CommonJS and Node.js compatible module loader licensed as open source under the LGPL v3. It aims to be
as lightweight as possible while not missing any features.

*Tarp.require is still work in progress. It is currently in a 'release candidate' state. This means that no
new features will be added until the release of version 1.0. The main work now is testing, bugfixing & refactoring.
Please be aware that some issues might slip through, until the test-routines are ready.*

## Features

* **Compatible** NodeJS 9.2.0 and CommonJS modules 1.1
* **Plain** No dependencies, no need to compile/bundle/etc the modules
* **Asynchronous** Non-blocking loading of module files
* **Modern** Makes use of promises and other features (support for older browsers via polyfills)
* **Lightweight** Just about 170 lines of code (incl. comments), minified version is about 1.7kB

## Installation

NPM and bower packages will be available once the first stable version has been released. For now just clone the
repository directly or add it to your git repository as a submodule:

```
$ git submodule add -b tarp https://github.com/letorbi/tarp.require.git
```

## Usage

Assuming you've installed Tarp.require in the folder *//example.com/tarp.require*, you only
have to add the following line to your HTML to load Tarp.require:

```
<script src"/tarp.require/require.min.js"></script>
```

Assuming you've loaded Tarp.require in *//example.com/page/index.html* and your main-module is located at
*//example.com/page/scripts/main.js*, you can use the following line to load the main-module:

```
Tarp.require("./scripts/main", true); // 'true' tells require to load the module asynchronously
```
It is strongly recommended to load the main module asynchrnously to prevent synchronous requests, which [are obsolete](https://xhr.spec.whatwg.org/#the-open()-method) by now. See the [synchronous and asynchronous loading section](#synchronous-and-asynchronous-loading) below for more details.

Inside any module you can use `require()` as you know it from CommonJS/NodeJS. Assuming you're in the main-module,
module-IDs will be resolved to the following paths:

  * `require("someMmodule")` loads *//example.com/page/node_modules/someModule.js*
  * `require("./someModule")` loads *//example.com/page/scripts/someModule.js*
  * `require("/someModule")` loads *//example.com/someModule.js*
  
#### Using `require()` directly from the HTML document

It is recommended that the only JavaScript in the HTML document is the line to load the main-module - any JavaScript of your page should be moved into the main-module to ensure that is is executed in a proper CommonJS/NodeJS environment.

However, if you really want to use `require()` directly in the HTML-document, you can add `Tarp.require` to the global scope:

```
self.require = Tarp.require;
```

## Synchronous and asynchronous loading

Tarp.require supports synchronous and asynchronous loading of modules. The default mode is synchronous loading to
ensure compatibility with NodeJS and CommonJS. However, asynchronous loading can be easily activated by adding `true` as
a second parameter:

```
// synchronous loading (can be pre-loaded asynchronously)
var someModule = require("someModule");
someModule.someFunc();

 // asynchronous loading
require("anotherModule", true).then(function(anotherModule) {
    anotherModule.anotherFunc();
});
```

Please note that thanks to the asynchronous pre-loading feature of Tarp.require, you will usually need the asynchronous
loading pattern only once per page to load the main-module.

### Asynchronous pre-loading

An asynchronous call of `require()` will not only try to load the module itself, but will also try to pre-load any
required submodules asynchronously. This way it is usually enough to load the main-module of a page asynchronously
and all other modules required for that page will be loaded asynchronously as well.

Pre-loading means, that even though the code of a submodule has been downloaded, it will not be executed, until the
require-call for that submodule is actually reached. So the execution-order is the same as if the modules were loaded
asynchronously.

Right now only plain require-calls are pre-loaded. This means that the ID of the module has to be one simple string.
Also require-calls with more than one parameter are ignored (since they are usually asynchronous calls by themselves).

**Example:** If *Module1* is required asynchronously and contains the require calls `require("Submodule1")`,
`require("Submodule2", true)` and `require("Submodule" + "3")` somewhere in its code, only *Submodule1* will be
pre-loaded, since the require-call for *Submodule2* has more than one parameter and the module-ID in the require-call
for *Submodule3* is not one simple string.

## Path resolving

Assuming Tarp.require has been loaded from *//example.com/page/index.html* and you are requiring a module directly
from the index-page, the module-IDs will be resolved to the following URLs:

 1. `someModule` resolves to *//example.com/page/node_modules/someModule.js*
 2. `./someModule` resolves to *//example.com/page/someModule.js*
 3. `/someModule` resolves to *//example.com/someModule.js*

So Tarp.require mainly resolves URLs in the same way as Node.js does resolve paths. The only difference is that
Tarp.require won't check *//example.com/node_modules/someModule.js*, if the module
file is not found at *//example.com/page/node_modules/someModule.js*, but will simply throw an error instead.

This decision has been made due to the fact that Tarp.require usually tries to load module files from a remote location
and searching through remote paths by requesting each probable location of a file would be very time-consuming.
Tarp.require relies on the server to resolve unknown files instead. The only occation then Tarp.require redirects a
request on its own is, when the module-ID points to a path that is redirected to a *package.json* file that contains a
`main` field.

#### Change the `node_modules` path globally

Add the following line **before** including Tarp.require into your page:

```
<script>var TarpConfig = { require: { paths: ['/path/to/node/modules'] } };</script>
```

### HTTP redirects

Tarp.require is able to handle temporary (301) and permanent (303) HTTP redirects. A common case where redirects might
be handy is to return the contents of *index.js* or *package.json* if an ID without a filename is requested. The following NGINX configuration rule will mimic the behavior of NodeJS:

```
location /node_modules {
    if ( -f $request_filename ) {
        break;
    }
    rewrite ^(.+).js$ $1 permanent; 
    if ( -f $request_filename/package.json ) {
        return 301 $request_uri/package.json;
    }
    if ( -f $request_filename/index.js ) {
        return 301 $request_uri/index.js;
    }
    return 404;
}
```

This will redirect all requests like */node_modules/someModule* to */node_modules/someModule/package.json*, if */node_modules/someModule* is a directory and if */node_modules/someModule/package.json* is a file. If that file doesn't exist, the request will be redirected to */node_modules/path/index.js*. If both files don't exist, a "404 Not Found" response will be sent. The rewrite rule in the fifth line ensures that a request like */node_modules/someModule.js* will be redirected to */node_modules/someModule* if the *.js* file doesn't exist.

### NPM packages

Tarp.require loads module-IDs specified the `main` field of a *package.json* file, if the following things are true:

 1. The *package.json* file is loaded via a redirect (like explained in the section above)
 2. The response contains a valid JSON object 
 3. The object has a property called `main`
 
If that is the case a second request will be triggered to load the modules specified in `main` and the exports of
that module will be returned. Otherwise simply the content of *package.json* is returned.

### The `module.paths` property

Tarp.require always uses the first item of the global `TarpConfig.require.paths` array to resolve an URL from a global
module-ID. Unlike Node.js it won't iterate over the whole array. Since the global config is always used, any change to `module.paths` won't affect the loding of modules.

Tarp.require also supports the `require.resolve.paths()` function that returns an array of paths that have been searched
to resolve the given module-ID.

----

Copyright 2013-2018 Torben Haase \<https://pixelsvsbytes.com>
