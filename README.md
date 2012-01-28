# Lightweight Asynchronous Error Handling v2 for Node.js (LAEH2)


The previous version of LAEH[1] is now deprecated. 

The reason is that is that some functions were removed, namely the support for Express.js and the MongoDB utility. This is because it is possible to nicely support Express.js while maintaining only a single version of the callback wrapper function. The MongoDB support is something that should reside in the `ultiz` package.
But the main reason is that the arguments of the `_x` function were swapped, which would silently break any LAEH-dependent code.

Changes from LAEH1:

* Only a single callback wrapper function, the `_x`.
* The `cb` and `chk` were moved to the front of `_x`'s argument list, to make code more readable.
* The `_x` function now nicely ties to error handling in Express.js and Connect.
* Lean Stacks support was also updated to be even more terse, and formatting options were added to support utilities which parse stack traces based on newlines (e.g. the error template in Express.js).


## Evolution

### 1. Unprotected callback code

```js
function someContext(arg, arg, callback) {

	someAsyncFunction(arg, arg, function(err, data) {
		// err is not checked but should be (a common case)
		throw new Error('fail'); // uncaught - will exit Node.js
	}

}
```

### 2. Manualy protected callback code, lots of clutter

```js
function someContext(arg, arg, callback) {

	someAsyncFunctionWithCallback(arg, arg, function(err, data) {
		if(err)
			callback(err);
		else {
			try {
				throw new Error('fail');
			}
			catch(e) {
				callback(e); // caught - return control manually
			}
		}
	}

}
```

### 3. LAEH2, an elegant solution

```js
function someContext(arg, arg, callback) {

	someAsyncFunctionWithCallback(arg, arg, _x(callback, true, function(err, data) {
		throw new Error('fail');
	}));
}
```

Parameter explanation:

* callback: in case of error return control to callback
* true: automatically check callback's err parameter and pass it directly to the parent callback if true
* function: the asynchronously executed callback function to wrap


### 4. Optional Goodies

LAEH2 stores the stacktrace of the thread that initiated the asynchronous operation which in turn called the callback. This stacktrace is then appended to the primary stacktrace of the error which was thrown in the callback, or the error which was passed to the callback by the asynchronous function.

LAEH2 then presents the stacktrace in a minified format, with optional hiding of frames of the `laeh2.js` itself, of the Node.js' core library files, shortens the often repeating string `/node_modules/` into `/$/`, and removes the current directory path prefix from the file names in the stacktrace.


## Usage

Install LAEH2:

	npm install laeh2

And then wrap your asynchronous callback with the `_x` function:

```js
var fs = require('fs');
var laeh = require('laeh2').leanStacks(true);
var _e = laeh._e; // optional
var _x = laeh._x;

var myfunc = function(param1, paramN, cb) {
	fs.readdir(__dirname, _x(cb, true, function(err, files) { // LINE #7
		// do your things here..
		_e('unexpected thing'); // throw your own errors, etc. LINE #9
	}));
}

myfunc('dummy', 'dummy', function(err) { // LINE #13
	if(err)
		console.log(err.stack);
});
```

This will print:
	
	unexpected thing < ./ex1.js(9) << ./ex1.js(7 < 13)
	
The async boundary is (by default) marked with `<<`.

If we disable `hiding` by passing `false` as the first parameter, the output will be something like:

	unexpected thing < /Users/ypocat/Github/laeh2/lib/laeh2.js(31) < ./ex1.js(9) < /Users/ypocat/Github/laeh2/lib/laeh2.js(56) << /Users/ypocat/Github/laeh2/lib/laeh2.js(45) < ./ex1.js(7 < 13) < module.js(432 < 450 < 351 < 310 < 470) < node.js(192)

If we enable hiding, and add some metadata:

```js
_e('unexpected thing', { msg: 'my metadata', xyz: 123 });
```

..the output, when configured with `.leanStacks(true, '\t')`, will be:

	unexpected thing < {
	        "msg": "my metadata",
	        "xyz": 123
	} ./ex2.js(9) << ./ex2.js(7 < 13)

And when configured with just `.leanStacks(true)`:

	unexpected thing < {"msg":"my metadata","xyz":123} ./ex3.js(9) << ./ex3.js(7 < 13)
	
This is a nice terse format which is also good when you store error messages to database or services like Loggly (with JSON input), as it saves a lot of space.

For comparison, this would be printed without using `.leanStacks`:

	Error: unexpected thing
	    at /Users/ypocat/Github/laeh2/lib/laeh2.js:31:8
	    at /Users/ypocat/Github/laeh2/example/ex4.js:9:3
	    at Object.oncomplete (/Users/ypocat/Github/laeh2/lib/laeh2.js:56:9)

The `leanStacks(hiding, prettyMeta)` call is optional, the `hiding` will hide stack frames from Node's core .js files and from `laeh2.js` itself. The `prettyMeta` is the third parameter for the `JSON.stringify` function, which is used to serialize your metadata objects (see below), and leaving it empty or null will serialize your metadata objects in-line.

Added in LAEH2 are 2 new parameters to `.leanStacks`: `frameSeparator` and `fiberSeparator`, which default to `' < '` and `' << '`, respectively. But if you use tools which rely on the newlines in your stack traces, you can set these accordingly, e.g. to `'\n'` and `'\n<<\n'`, respectively, e.g. `.leanStacks(true, null, '\n', '\n<<\n')`:

	unexpected thing
	./ex6.js(9)
	<<
	./ex6.js(7 < 13)

### Express.js

When coding handlers or params for Express.js or Connect, just pass the `next` parameter as the eventual callback, e.g.:

```js
app.param('reg', function(req, res, next, email) {
	db.hgetall('reg:' + email, _x(next, true, function(err, reg) {
		if(!reg.lickey)
			return next('No such registration');
		req.reg = reg;
		next();
	}));
});
```

Now any error thrown in the callback called by Redis' `hgetall` will be captured and passed to the `next()` function. Likewise, should Redis respond with an error passed via the `err` parameter, this parameter is automatically checked and the error will be passed to the `next()` function. Easy peasy LAEH squeezy.

Note: There is no need to `_x`-wrap the callback passed to the `app.param()` call (or `app.get() etc.`), as Express.js handles wraps and handles this first level automatically.

### Other

The `_e(err, meta)` function is just a convenient error checking, wrapping and throwing. E.g. `_e('something')` will throw `new Error('something')` and `_e(null)` will not do anything. The `meta` parameter is an optional accompanying information for the error to be thrown, which is then displayed when you let LAEH to display your errors using the `leanStacks()` call.

In the `_x(cb, chk, func)`, the func is you callback to be wrapped. If it follows the node convention of `func(err, args)`, you can pass `chk` as true, which will automatically check for the `err` to be null, and call the eventual callback if it isn't null. The eventual callback is passed as the `cb` argument, or if omitted, it is tried to be derived from the last argument passed to the function you are wrapping, e.g. if the signature is `func(err, args, cb)`, the `cb` is taken from its arguments.
