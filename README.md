# express-validator

[![npm version](https://img.shields.io/npm/v/express-validator.svg)](https://www.npmjs.com/package/express-validator)
[![Build Status](https://img.shields.io/travis/ctavan/express-validator.svg)](http://travis-ci.org/ctavan/express-validator)
[![Dependency Status](https://img.shields.io/david/ctavan/express-validator.svg)](https://david-dm.org/ctavan/express-validator)
[![Coverage Status](https://img.shields.io/coveralls/ctavan/express-validator.svg)](https://coveralls.io/github/ctavan/express-validator?branch=master)

An [express.js]( https://github.com/visionmedia/express ) middleware for
[node-validator]( https://github.com/chriso/validator.js ).

## Installation

```
npm install express-validator
```

## Usage

```javascript
var util = require('util'),
    bodyParser = require('body-parser'),
    express = require('express'),
    expressValidator = require('express-validator'),
    app = express();

app.use(bodyParser.json());
app.use(expressValidator([options])); // this line must be immediately after any of the bodyParser middlewares!

app.post('/:urlparam', function(req, res) {

  // VALIDATION
  // checkBody only checks req.body; none of the other req parameters
  // Similarly checkParams only checks in req.params (URL params) and
  // checkQuery only checks req.query (GET params).
  req.checkBody('postparam', 'Invalid postparam').notEmpty().isInt();
  req.checkParams('urlparam', 'Invalid urlparam').isAlpha();
  req.checkQuery('getparam', 'Invalid getparam').isInt();

  // OR assert can be used to check on all 3 types of params.
  // req.assert('postparam', 'Invalid postparam').notEmpty().isInt();
  // req.assert('urlparam', 'Invalid urlparam').isAlpha();
  // req.assert('getparam', 'Invalid getparam').isInt();

  // SANITIZATION
  // as with validation these will only validate the corresponding
  // request object
  req.sanitizeBody('postparam').toBoolean();
  req.sanitizeParams('urlparam').toBoolean();
  req.sanitizeQuery('getparam').toBoolean();

  // OR find the relevent param in all areas
  req.sanitize('postparam').toBoolean();

  // Alternatively use `var errors = yield req.getValidationErrors();`
  // when using generators e.g. with co-express
  req.getValidationErrors().then(function(errors) {
    if (errors) {
      res.send('There have been validation errors: ' + util.inspect(errors), 400);
      return;
    }
    res.json({
      urlparam: req.params.urlparam,
      getparam: req.params.getparam,
      postparam: req.params.postparam
    });
  });
});

app.listen(8888);
```

Which will result in:

```
$ curl -d 'postparam=1' http://localhost:8888/test?getparam=1
{"urlparam":"test","getparam":"1","postparam":true}

$ curl -d 'postparam=1' http://localhost:8888/t1est?getparam=1
There have been validation errors: [
  { param: 'urlparam', msg: 'Invalid urlparam', value: 't1est' } ]

$ curl -d 'postparam=1' http://localhost:8888/t1est?getparam=1ab
There have been validation errors: [
  { param: 'getparam', msg: 'Invalid getparam', value: '1ab' },
  { param: 'urlparam', msg: 'Invalid urlparam', value: 't1est' } ]

$ curl http://localhost:8888/test?getparam=1&postparam=1
There have been validation errors: [
  { param: 'postparam', msg: 'Invalid postparam', value: undefined} ]
```

### Middleware Options
####`errorFormatter`
_function(param,msg,value)_

The `errorFormatter` option can be used to specify a function that can be used to format the objects that populate the error array that is returned in `req.getValidationErrors()`. It should return an `Object` that has `param`, `msg`, and `value` keys defined.

```javascript
// In this example, the formParam value is going to get morphed into form body format useful for printing.
app.use(expressValidator({
  errorFormatter: function(param, msg, value) {
      var namespace = param.split('.')
      , root    = namespace.shift()
      , formParam = root;

    while(namespace.length) {
      formParam += '[' + namespace.shift() + ']';
    }
    return {
      param : formParam,
      msg   : msg,
      value : value
    };
  }
}));
```

####`customValidators`
_{ "validatorName": function(value, [additional arguments]), ... }_


The `customValidators` option can be used to add additional validation methods as needed. This option should be an `Object` defining the validator names and associated validation functions.

Define your custom validators:

```javascript
app.use(expressValidator({
 customValidators: {
    isArray: function(value) {
        return Array.isArray(value);
    },
    gte: function(param, num) {
        return param >= num;
    }
 }
}));
```
Use them with their validator name:
```javascript
req.checkBody('users', 'Users must be an array').isArray();
req.checkQuery('time', 'Time must be an integer great than or equal to 5').isInt().gte(5)
```
####`customSanitizers`
_{ "sanitizerName": function(value, [additional arguments]), ... }_

The `customSanitizers` option can be used to add additional sanitizers methods as needed. This option should be an `Object` defining the sanitizer names and associated functions.

Define your custom sanitizers:

```javascript
app.use(expressValidator({
 customSanitizers: {
    toSanitizeSomehow: function(value) {
        var newValue = value;//some operations
        return newValue;
    },
 }
}));
```
Use them with their sanitizer name:
```javascript
req.sanitize('address').toSanitizeSomehow();
```

## Validation

#### req.check();
```javascript
   req.check('testparam', 'Error Message').notEmpty().isInt();
   req.check('testparam.child', 'Error Message').isInt(); // find nested params
   req.check(['testparam', 'child'], 'Error Message').isInt(); // find nested params
```

Starts the validation of the specifed parameter, will look for the parameter in `req` in the order `params`, `query`, `body`, then validate, you can use 'dot-notation' or an array to access nested values.

If a validator takes in params, you would call it like `req.assert('reqParam').contains('thisString');`.

Validators are appended and can be chained. See [chriso/validator.js](https://github.com/chriso/validator.js) for available validators, or [add your own](#customvalidators).

#### req.assert();
Alias for [req.check()](#reqcheck).

#### req.validate();
Alias for [req.check()](#reqcheck).

#### req.checkBody();
Same as [req.check()](#reqcheck), but only looks in `req.body`.

#### req.checkQuery();
Same as [req.check()](#reqcheck), but only looks in `req.query`.

#### req.checkParams();
Same as [req.check()](#reqcheck), but only looks in `req.params`.

#### req.checkHeaders();
Only checks `req.headers`. This method is not covered by the general `req.check()`.

#### req.checkCookies();
Only checks `req.cookies`. This method is not covered by the general `req.check()`.

## Validation by Schema

Alternatively you can define all your validations at once using a simple schema.
Schema validation will be used if you pass an object to any of the validator methods.

You may pass per-validator error messages with the `errorMessage` key.  
Validator options may be passed via `options` key as an array when various values are needed,
or as a single non-null value otherwise.

```javascript
req.checkBody({
 'email': {
    optional: {
      options: { checkFalsy: true } // or: [{ checkFalsy: true }]
    },
    isEmail: {
      errorMessage: 'Invalid Email'
    }
  },
  'password': {
    notEmpty: true,
    matches: {
      options: ['example', 'i'] // pass options to the validator with the options property as an array
      // options: [/example/i] // matches also accepts the full expression in the first parameter
    },
    errorMessage: 'Invalid Password' // Error message for the parameter
  },
  'name.first': { //
    optional: true, // won't validate if field is empty
    isLength: {
      options: [{ min: 2, max: 10 }],
      errorMessage: 'Must be between 2 and 10 chars long' // Error message for the validator, takes precedent over parameter message
    },
    errorMessage: 'Invalid First Name'
  }
});
```

You can also define a specific location to validate against in the schema by adding `in` parameter as shown below:

```javascript
req.check({
 'email': {
    in: 'query',
    notEmpty: true,
    isEmail: {
      errorMessage: 'Invalid Email'
    }
  }
});
```

Please remember that the `in` attribute will have always highest priority. This mean if you use `in: 'query'` then checkQuery() will be called inside even if you do `checkParams()` or `checkBody()`. For example, all of these calls will check query params for email param:


```javascript
var schema = {
 'email': {
    in: 'query',
    notEmpty: true,
    isEmail: {
      errorMessage: 'Invalid Email'
    }
  },
  'password': {
    notEmpty: true,
    matches: {
      options: ['example', 'i'] // pass options to the validator with the options property as an array
      // options: [/example/i] // matches also accepts the full expression in the first parameter
    },
    errorMessage: 'Invalid Password' // Error message for the parameter
  }
};

req.check(schema);        // will check 'password' no matter where it is but 'email' in query params
req.checkQuery(schema);   // will check 'password' and 'email' in query params
req.checkBody(schema);    // will check 'password' in body but 'email' in query params
req.checkParams(schema);  // will check 'password' in path params but 'email' in query params
```

Currently supported location are `'body', 'params', 'query'`. If you provide a location parameter that is not supported, the validation process for current parameter will be skipped.

## Validation errors

You have two choices for getting validation errors:

```javascript
req.assert('email', 'required').notEmpty();
req.assert('email', 'valid email required').isEmail();
req.assert('password', '6 to 20 characters required').len(6, 20);

req.getValidationErrors(); // Basic errors
req.getValidationErrors(true); // Mapped Errors
```

Basic Errors:

```javascript
[
  {param: "email", msg: "required", value: "<received input>"},
  {param: "email", msg: "valid email required", value: "<received input>"},
  {param: "password", msg: "6 to 20 characters required", value: "<received input>"}
]
```

Mapped Errors:

```javascript
{
  email: {
    param: "email",
    msg: "valid email required",
    value: "<received input>"
  },
  password: {
    param: "password",
    msg: "6 to 20 characters required",
    value: "<received input>"
  }
}
```
*Note: Using mapped errors will only provide the last error per param in the chain of validation errors.*

### String formatting for error messages

Error messages can be customized to include both the value provided by the user, as well as the value of any parameters passed to the validation function, using a standard string replacement format:

`%0` is replaced with user input  
`%1` is replaced with the first parameter to the validator  
`%2` is replaced with the second parameter to the validator  
etc...

Example:
```javascript
req.assert('number', '%0 is not an integer').isInt();
req.assert('number', '%0 is not divisible by %1').isDivisibleBy(5);
```

*Note:* string replacement does **not** work with the `.withMessage()` syntax. If you'd like to have per-validator error messages with string formatting, please use the [Validation by Schema](#validation-by-schema) method instead.

### Per-validation messages

You can provide an error message for a single validation with `.withMessage()`. This can be chained with the rest of your validation, and if you don't use it for one of the validations then it will fall back to the default.

```javascript
req.assert('email', 'Invalid email')
    .notEmpty().withMessage('Email is required')
    .isEmail();

req.getValidationErrors()
   .then(function(errors){
     // do something with errors
   });

```
errors:

```javascript
[
  {param: 'email', msg: 'Email is required', value: '<received input>'}
  {param: 'email', msg: 'Invalid Email', value: '<received input>'}
]
```

## Optional input

You can use the `optional()` method to skip validation. By default, it only skips validation if the key does not exist on the request object. If you want to skip validation based on the property being falsy (null, undefined, etc), you can pass in `{ checkFalsy: true }`.

```javascript
req.checkBody('email').optional().isEmail();
//if there is no error, req.body.email is either undefined or a valid mail.
```

## Sanitizer

#### req.sanitize();
```javascript

req.body.comment = 'a <span>comment</span>';
req.body.username = '   a user    ';

req.sanitize('comment').escape(); // returns 'a &lt;span&gt;comment&lt;/span&gt;'
req.sanitize('username').trim(); // returns 'a user'

console.log(req.body.comment); // 'a &lt;span&gt;comment&lt;/span&gt;'
console.log(req.body.username); // 'a user'

```

Sanitizes the specified parameter (using 'dot-notation' or array), the parameter will be updated to the sanitized result. Cannot be chained, and will return the result. See [chriso/validator.js](https://github.com/chriso/validator.js) for available sanitizers, or [add your own](#customsanitizers).

If a sanitizer takes in params, you would call it like `req.sanitize('reqParam').whitelist(['a', 'b', 'c']);`.

If the parameter is present in multiple places with the same name e.g. `req.params.comment` & `req.query.comment`, they will all be sanitized.

#### req.filter();
Alias for [req.sanitize()](#reqsanitize).

#### req.sanitizeBody();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.body`.

#### req.sanitizeQuery();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.query`.

#### req.sanitizeParams();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.params`.

#### req.sanitizeHeaders();
Only sanitizes `req.headers`. This method is not covered by the general `req.sanitize()`.

#### req.sanitizeCookies();
Only sanitizes `req.cookies`. This method is not covered by the general `req.sanitize()`.

### Regex routes

Express allows you to define regex routes like:

```javascript
app.get(/\/test(\d+)/, function() {});
```

You can validate the extracted matches like this:

```javascript
req.assert(0, 'Not a three-digit integer.').len(3, 3).isInt();
```

## Deprecated methods

Express Validator previously recommended using `req.validationErrors()` and
`req.asyncValidationErrors()` which have now been deprecated in favour of
`req.getValidationErrors()`.

## Changelog

See [CHANGELOG.md](CHANGELOG.md)

## Contributors

- Christoph Tavan <dev@tavan.de> - Wrap the gist in an npm package
- @orfaust - Add `validationErrors()` and nested field support
- @zero21xxx - Added `checkBody` function

## License

Copyright (c) 2010 Chris O'Hara <cohara87@gmail.com>, MIT License
