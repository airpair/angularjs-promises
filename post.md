AngularJS promises are an extremely powerful tool. They allow you to make a multi-layered and complex system based on asynchronous functions, with error and in-progress notification handling, all without getting into callback hell.

This post attempts to explain both creating and using AngularJS promises. It assumes some familiariy with AngularJS, specifically defining and injecting services and using controllers.

### What are AngularJS promises?

Promises are Javascript objects with `then` and `finally` functions:

```javascript
{
  'then': function(successCallback, errorCallback, notifyCallback) {
    // Black box code
  },
  'finally': function(finallyCallback) {
    // Black box code
  }
}
```

and they are returned from functions whose meaningful result will be found or calculated later, i.e. asynchronously. The function will choose which of the callbacks to call, depending on success or failure of the asynchronous action. In case of success, the `successCallback` is called, and the promise is said to be _resolved_ with a result. In case of failure, the `errorCallback` is called, and the promise is said to have been _rejected_ with an error, or sometimes called a reason.

You should note:

* You don't ever need to construct a promise yourself using object notation. As will be discussed, they will be created via function calls.
* Angular promises have other functions, but they just shortcuts, so are ommited for this post.

### Using promises

A good first step in understanding promises is to use ones that are created by an existing service. A common use of a promise in AngularJS, are ones created by the `$http` service.

```javascript
var promise = $http.get('/my-url');
promise.then(function(result) {
  // Do something with the results of the GET request
});
````

In the preceeding example, the success of the GET request will cause the `$http` service to _resolve_ the promise with the result of the GET request. This will then call the `successCallback` passing this resolved value as the first parameter.

It's quite common to not assign promises to an intermediate variable. The following is equivalent to the preceding example.

```javascript
$http.get('/my-url').then(function(result) {
  // Do something with the results of the GET request
});
```

In the above examples, only the `successCallback` is used from the promise. If we want to handle a failure then

```javascript
$http.get('/my-url-that-might-fail').then(function(result) {
  // Do something with the result of the GET request if it succeeds
}, function(error) {
  // Do something with the error if it fails
});
```

Although not currently possible, if `$http` ever ends up being modified to give in-progress notfications, then this could be used as

```javascript
$http.get('/my-url').then(function(result) {
  // Do something with the results of the GET request
}, function(error) {
  // Do something with the error
}, function(update) {
  // Do something with the update
});
```

If you want to perform some action after the `$http` request has completed, whether it succeeded or failed, then you can use the `finally` function.

```javascript
$http.get('/my-url').finally(function() {
  // Do something after either success or failure
});
```

The success, failure, notify callbacks can all be ommited, or passed `null` if you have no action to be performed

### Chaining Promises

The real power from promises comes from chaining. The first key to understanding this is

> The `then` and `finally` functions each return a new promise, known as a _derived_ promise.

An example of using derived promises:

```javascript
var promiseA = $http.get('/my-url');
var promiseB = promiseA.then(function(result) {
  // Do something with the results of the GET request
});
promiseB.then(function(result) {
  // Do something with the resolved result of promiseB
})
```

As with the case of a single promise, it is quite common to not assign the derived promises to intermediate variables. The following is equivalent to the preceeding example:

```javascript
$http.get('/my-url').then(function(result) {
  // Do something with the result of the GET request
}).then(function(results) {
  // Do something with the resolved result of promiseB
})
```

The second key to understanding chains are:

> Derived promises are resolved/rejected with the returned resolved/rejected value of the callback that was run.

In practice, this means there are 3 possible ways of controlling the derived promise. It can be resolved immediately, it can be rejected immediately, or its own resolution/rejection can be deferred further until a 3rd promise has been resolved/rejected, in which case the derived promise is resolved/rejected with the 3rd promise's resolved/rejected value.

### Resolving a derived promise immediately

If you return anything but a promise from the callback that is run, the derived promise will be resolved immediately with that returned value

```javascript
$http.get('/my-url').then(function(result) {
  return 'my-immediate-value';
}).then(function(results) {
  // results === 'my-immediate-value';
})
```

Be aware that not explicty returning a value means that you have returned `undefined`, and the derived promise will be resolved with `undefined`.

```javascript
$http.get('/my-url').then(function(result) {
  // Not returning a value.
}).then(function(results) {
  // results === undefined
})
```

The above applies to both the success and the error callbacks. Returning any non-promise value from the error callback means that the derived promise will be _resolved_, and not rejected:

```javascript
$http.get('/my-url-that-does-not-exist').then(function(results) {
}, function(error) {
  return 'my-immediate-value';
}).then(function(results) {
  // results === 'my-immediate-value'
})
```

As with the success callback, not returning a value from the error callback means the derived promise will be resolved with `undefined`

```javascript
$http.get('/my-url-that-does-not-exist').then(function(result) {
}, function(error) {
  // Not returning a value.
}).then(function(results) {
  // results === undefined
})
```

I've done the above accidentally: it derives a resolved promise from a rejected one, which might be be desirable.

### Rejecting a derived promise immediately

You may want to _reject_ a derived promise, even if the original promise was resolved. This is done by returning the result of `$q.reject()`.

```javascript
$http.get('/my-url').then(function(result) {
  return $q.reject('my-failure-reason');
}).then(function(results) {
  // The code never gets here
}, function(error) {
  // error === 'my-failure-reason'
});
```

If you want to then fail the derived promise from an error callback, then you can do the same thing in it:

```javascript
$http.get('/my-url-that does not exist').then(function(results) {
  // The code never gets here if the GET was unsuccessfull
}, function(error) {
   return $q.reject('my-failure-reason');
}.then(function() {
  // The code never gets here if the GET was unsuccessfull
}, function(error) {
  // error === 'my-failure-reason'
});
```

### Deferring a derived promise

A powerful aspect of derived promises is that their resolution/rejection can be deferred until another promise has been resolved/rejected. This is done by returning a promise from the success or error callback. For example, if you want to run `$http.get` calls sequentially, and then do something after the final is successful, you can do this by returning the result of `$http.get` from the callback:

```javascript
$http.get('/my-first-url').then(function(results) {
  return $http.get('/my-second-url')
}).then(function(results) {
  // results here are the results of the GET to /my-second-url 
});
```

Because each `then` call again returns a promise, you can easily add to this chain:

```javascript
$http.get('/my-first-url').then(function(results) {
  return $http.get('/my-second-url')
}).then(function(results) {
  return $http.get('/my-third-url')
}).then(function(results) {
  // results here are the results of the GET to /my-third-url 
});
```

### Rejection/error handling in promise chains

If a promise is rejected, then every subsequent promise in the chain will be rejected, until one is reached with an error callback.

```javascript
$http.get('/my-first-url-that-fails').then(function(results) {
  // Never called
  return $http.get('/my-second-url')
}).then(function(results) {
  // Never called
  return $http.get('/my-third-url')
}).then(function(results) {
  // results here are the results of the GET to /my-third-url 
}, function(error) {
  // Error callback called
});
```

When specifying an error callback, be careful what you return. If you return a non-promise value, which includes `undefined` by not specifying a return value, then the derived promise for that callback will be _resolved_, and not rejected.

```javascript
$http.get('/my-first-url-that-fails').then(function(result) {
  // Never called
  return $http.get('/my-second-url')
}, function(error) {
  // Error callback called
}).then(function(results) {
  // This *is* called, because the previous
  // callback returned undefined
  return $http.get('/my-third-url')
});
```

### Layered APIs

A common use of promises is chaining them via layered APIs. A typical pattern in AngularJS is to have calls to `$http` functions in a service, so controllers are not aware that `$http` is used.

    MyController -> MyService -> $http

You can do this using a structure like:

```javascript
// In MyService
this.fetchResults = function() {
  return $http.get('/my-url');
};

// In MyController
$scope.fetchResults = function() {
  MyService.fetchResults().then(function(results) {
    // Do something with results
  });
}
```

However, this means that the controller will be exposed to HTTP headers and statuses. To hide this lower-level detail, you can add post-processing in the service via a derived promises:

```javascript
this.fetchResults = function() {
  return $http.get('/my-url').then(function(results) {
    // Just return the HTTP body
    return results.data;
  );
};
```

You can also include some of your own error handling, so in the case of a failed request, the controller can be ignorant of any details of HTTP:

```javascript
// In MyService
this.fetchResults = function() {
  return $http.get('/my-url').then(function(results) {
    // Just return the http body
    return results.data;
), function(error) {
  return $q.reject('Oh no!');
});

// In MyController
$scope.fetchResults = function() {
  MyService.fetchResults().then(function(results) {
    // Do something with results
  }, function(error) {
    // Do something with error if it occurs
    // which would be 'Oh no!'
  });
}
```

### Creating Promises

If you're not chaining onto an existing promise, you might need to create a new, non-derived, promise. You can do this by calling `$q.defer()`. This returns a _deferred_ object:

```javascript
var deferred = $q.defer();
``` 

The deferred object contains a promise, and methods to call to control that promise:

```javascript
{
  'resolve': function(result) {
    // Black box code
  },
  'reject': function(error) {
    // Black box code
  },
  'notify': function(update) {
    // Black box code
  },
  'promise': // Promise as described above
}
```

When you want to resolve the promise, you can call the `resolve` function. Similarly for the `reject` and `notify` functions.

Using this, a simple timer (ignoring the existence of `$timeout`) can be written as

```javascript
var simpleTimeout = function(time) {
  var deferred = $q.defer();
  $window.setTimeout(function() {
    deferred.resolve('Done!');
  }, time);
  return deferred.promise;
}
```

Which could be used as:

```javascript
simpleTimeout(1000).then(function(result) {
  // Code gets here after 1 second, and
  // result === 'Done!'
});
```

You could also have a timer that sends a notification:

```javascript
var simpleTimeout = function(time) {
  var deferred = $q.defer();

  $window.setTimeout(function() {
    deferred.resolve('Done!');
  }, time);$

  $window.setTimeout(function() {
    deferred.notify('Half way there!');
  }, time/2);

  return deferred.promise;
}
```

Which could be use as:

```javascript
simpleTimeout(1000).then(function(result) {
  // Code gets here after 1 second, and
  // result === 'Done!'
}, null, function(update) {
  // After 1/2 second, code gets here
  // and update === 'Half way there!'
)};
```

### Exceptions thrown in callbacks

A not very documented feature of Angular promises is that when exceptions are thrown in the callbacks, via `throw`, then the derived promise will be rejected with that thrown value.

```javascript
$http.get('/my-url').then(function(results) {
  throw 'my-failure-reason';
}).then(function(results) {
  // The code never gets here
}, function(error) {
  // error === 'my-failure-reason'
});
```

This will also trigger Angular's registered exception handler, so it's not quite equivalent to using `$q.reject()`.