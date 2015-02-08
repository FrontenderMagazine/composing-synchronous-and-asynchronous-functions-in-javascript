# Composing Synchronous and Asynchronous Functions in JavaScript

Our example application implements a function `createEmployee` that is used to
create an employee from a personId.

To create an employee, our system loads some data from our database, validates
that data, and then performs an insert. Some of our functions are written in 
continuation-passing style (they accept a callback) and some are written in a 
direct style (they return values and/or throw exceptions). We’d like to compose 
these functions in such a way that they succeed or fail as a single unit – that 
any error in any segment of the sequence will cause subsequent steps to be 
skipped with a failure – but with some of our validations happening 
synchronously and some asynchronously, this can be difficult to do.

To work around this problem, programmers will inline anonymous functions to
thread return values and exceptions from direct-style code to our callbacks. For
example:

    function hasWildHair(person) {
        return person.hairColor !== 'Green' || person.hairColor !== 'Pink'
    }
    
    function isOfAge(person) {
        return person.age > 17;
    }
    
    function ensureDriversLicense(person, callback) {
        Database.getDriversLicenseByPersonId(person.id, function(err, license) {
          if (err) {
              callback(err);
          }
          else if (license.expired) {
              callback('Person must have valid license.');
          }
          else {
              callback(null, person);
          }
        });
    }
    
    function createEmployee(personId, callback) {
        var workflow = async.compose(
            Database.createEmployee,
            ensureDriversLicense,
            ensureWashingtonAddress,
            function(person, callback) {
                if (hasWildHair(person)) {
                    callback(null, person);   
                }
                else {
                    callback('Person must have wild hair.');
                }
            },
            function(person, callback) {
                if (isOfAge(person)) {
                    callback(null, person);
                }
                else {
                    callback('Person must be 18+ years of age.');
                }
            }
            Database.getPersonById);
        
        workflow(personId, callback);
    }
    

There are a few problems with this approach:

1.  We’ve introduced two throwaway function expressions which add visual
    noise and some maintenance overhead.
   
2.  Our direct-style code (hair color and of-age validations) isn’t
    accessible outside of the CPS function that wraps it.
   
3.  Our throwaway functions contain the logic to transmute the predicate-
    failures to error messages. Check for wild hair in ten places in your codebase? 
    Ten places to add the check for null and hitting a callback with ‘Person must 
    have wild hair.
    ’

In this post, I’ll demonstrate an approach to using higher-order functions to
lift direct-style functions into the world of callbacks – enabling composition 
without the introduction of boilerplate function expressions. Once done, the 
above code will look like this:

    var createEmployee = async.compose(
        Database.createEmployee,
        ensureDriversLicense,
        ensureWashingtonAddress,
        asyncify(ensureWildHair),
        asyncify(ensureOfAge),
        Database.getPersonById);
    

### Step 1: Writing Failable Functions

First, we’ll write a higher-order function that accepts a predicate, a value
to which we’ll apply the predicate, and an error to throw in the event that our 
value does not satisfy the predicate. Why throw an error? This helps consuming 
functions differentiate the function’s success from failure.

    function ensure(predicate, error, value) {
        if (predicate(value) {
            return value;
        }
        else {
            throw error;
        }
    }
    

We can now compose `ensure` with our predicates, creating reusable failable
validators:

    var ensureWildHair = _.partial(ensure, hasWildHair, 'Person must have wild hair.');
    var ensureOfAge    = _.partial(ensure, ofAge, 'Person must be 18+ years of age.');
    

…which moves some of the error handling out of our larger, employee-creation
function:

    function createEmployee(personId, callback) {
        var workflow = async.compose(
            Database.createEmployee,
            ensureDriversLicense,
            ensureWashingtonAddress,
            function(person, callback) {
                try {
                    callback(null, ensureHasWildHair(person));
                }
                catch (e) {
                    callback(e);
                }
            },
            function(person, callback) {
                try {
                    callback(null, ensureOfAge(person));
                }
                catch (e) {
                    callback(e);
                }
            }
            Database.getPersonById);
        
        workflow(personId, callback);
    }
    

### Step 2: Working in the World of Callbacks

We’ve dumbed down the responsibilities of the throwaway function expressions
and centralized error creation and predicate handling into a generalized utility
function`ensure`. Now, we’ll write some code that will allow us to use a
direct-style function in a continuation-passing style context.

    function asyncify(f) {
      return function _asyncify() {
        var args = Array.prototype.slice.call(arguments, 0);
        var argsWithoutLast = args.slice(0, -1);
        var callback = args[args.length-1];
        var result, error;
    
        try {
          result = f.apply(this, argsWithoutLast);
        }
        catch (e) {
          error = e;
        }
    
        setTimeout(function() {
          callback(error, result);
        }, 0);
      }
    }
    

This new function allows us to map any n-ary function that either returns a
value or throws an error to a (n+1)-ary function whose last argument is expected
to be a Node.js-style callback whose first argument is the thrown error (if it 
exists) and second argument will be the returned value (if we didn’t throw
).

An example:

    function head(xs) {
        if (_.isArray(xs) && !_.isEmpty(xs)) {
            return xs[0];
        }
        else {
            throw "Can't get head of empty array.";
        }
    }
    
    var cbHead = asyncify(head);
    
    cbHead([1, 2, 3], function(err, result) {
        // result === 1
    });
    
    cbHead([], function(err, result) {
        // err === "Can't get head of empty array.";
    });
    

We can now use `asyncify` function to reshape our direct-style functions into
functions that accept callbacks.

    function createEmployee(personId, callback) {
        var workflow = async.compose(
            Database.createEmployee,
            ensureDriversLicense,
            ensureWashingtonAddress,
            asyncify(ensureHasWildHair),
            asyncify(ensureOfAge),
            Database.getPersonById);
        
        workflow(personId, callback);
    }
    

Finally, we can eliminate the function expression completely; we can instead
define`createEmployee` as the composition of our other functions. Since the
expression does nothing more than delegate to a function with the same signature,
we can safely eliminate it.

    var createEmployee = async.compose(
        Database.createEmployee,
        ensureDriversLicense,
        ensureWashingtonAddress,        
        asyncify(ensureHasWildHair), 
        asyncify(ensureOfAge),
        Database.getPersonById);
    

### In Summary

Our final implementation reduce the code bespoke code required to validate and
create an employee to an absolute minimum. Our resulting application is highly 
modular;`ensure` and `asyncify` allow the bits to be used in a variety of
contexts outside of`createEmployee`. In the end, we’ve generalized things to
the point where our job is to simply compose generalized functions together to 
create something specific to the problem at hand.

### Related Reads (that I didn’t write)

### Related Reads (that I did)

### Feedback

  Comments: 5<section id="respond" class="append-xs-30">

### Your feedback

[Click here to cancel reply.][1]


 [1]: http://blog.carbonfive.com/2015/01/29/composing-synchronous-and-asynchronous-functions-in-javascript/#respond