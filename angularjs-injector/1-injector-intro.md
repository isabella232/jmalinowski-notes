# AngularJS $injector intro

Quick note: AngularJS refers to version 1.x, Angular refers to version 2+. This is official naming.

$injector is a part of AngularJS which provides an instance or value of something by name.

```javascript
const MyService = $injector.get('MyService');
MyService.getValue();
```

However, this is not how `$injector` is typically used. Where $injector usually does the hard work is:
```javascript
angular.module('MyApp').factory('myFactory', function ($http) {
  return {
    fetchSomething: () => $http.get('/something');
  };
});
```

What happened here? So *almost any* function you give AngularJS will take injected arguments as parameters. For more info go to the [annotation section](#annotating-dependencies-to-inject).

Another way to see that piece of code is:
```javascript
angular.module('MyApp').factory('myFactory', function ($injector) {
  // note that in previous example this line would be executed
  // before the function is executed
  const $http = $injector.get('$http');

  return {
    fetchSomething: () => $http.get('/something');
  };
});
```

## API

**Important caveat: these are simplified for the purpose of the tutorial.**

### angular.module()
```typescript
angular.module(name: string, dependencies?: string[]): AngularJSModule
```

There's a difference between:
```javascript
// returns the singleton instance of the module
angular.module('MyApp')
```
and
```javascript
// initializes the module and returns its singleton instance
angular.module('MyApp', ['ourOtherModule'])
```

Also this is totally fine (Haskel programmers would call it idempotent):
```javascript
angular.module('MyApp');
angular.module('MyApp');
```

And this is not fine:
```javascript
angular.module('MyApp', ['ourOtherModule']);
angular.module('MyApp', ['ourOtherModule']); // error thrown here
```
And it's there to prevent you from shooting yourself in the foot. Consider what would happen if you called it with different `dependencies` values.

### angular.factory()
[Official TS typing](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/angular/index.d.ts#L257)
```typescript
factory(name: string, fn: function | any[]): AngularJSModule;
```
If I could, I would restrict `fn` to say it's an array of string, except for its last item, which is a function. I don't think there's a way to say that in TypeScript.

So this takes into consideration 3 ways of injecting, which we'll see in action in [annotation section](#annotating-dependencies-to-inject).

### $injector.get()
[Official TS typing](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/angular/index.d.ts#L2203)

`caller` is for creating paths when Dependency Injection fails, like `Unknown provider: Service1 -> Service2 -> Service3`. [See this for implementation](https://github.com/angular/angular.js/blob/master/src/auto/injector.js#L724).

## Annotating dependencies to inject

We've already seen:
```javascript
angular.module('MyApp').factory('myFactory', function ($http) {
  return {
    fetchSomething: () => $http.get('/something');
  };
});
```

So an obvious question is, how will AngularJS know to pass an instance of `$http` to our factory function.

```javascript
const myFactory = function ($http) {
  return {
    fetchSomething: () => $http.get('/something');
  };
});
console.log(myFactory.toString());
//  will show this in the console:
//  function ($http) {
//    return {
//      fetchSomething: () => $http.get('/something');
//    };
//  });
```

Yes, it's not a mistake, in ECMAScript casting a function to string gives its contents. If we apply a Regex on this string and look for something like `/function\((\s+\,)*\)/`, then we'll get a list of names of things to inject.

AngularJS implements that in a more thorough way, [see it on github starting on line 109](https://github.com/angular/angular.js/blob/master/src/auto/injector.js#L109).

This leads to an issue - usually during front-end code "compilation", names of local variables are shortened to make resulting package smaller. So our function will now be:
```javascript
const myFactory = function (a) {
  return {
    fetchSomething: () => a.get('/something');
  };
});
console.log(myFactory.toString());
//  will show this in the console:
//  function (a) {
//    return {
//      fetchSomething: () => a.get('/something');
//    };
//  });
```

Which is an issue. Of course we now can't extract information about requested dependencies anymore. There's 2 ways to solve it:
```javascript
myFactoryWithInject.$inject = ['$http'];
function myFactoryWithInject (a) {
  return {
    fetchSomething: () => a.get('/something');
  };
});
```
See [this article on MDN](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting) for why we can assign a field to a function before it's declaration. And I purposefully used it like that so it's not a suprise one day when you find it written this way.

This case is handled [on this line](https://github.com/angular/angular.js/blob/master/src/auto/injector.js#L99), but it's not very readable. Value of `$inject` is taken from `fn.$inject`, and if present, execution skips to the end of `function annotate()` where it's returned.

The second way is:
```javascript
const myFactory = ['$http', function (a) {
  return {
    fetchSomething: () => a.get('/something');
  };
}];
angular.module('MyApp').factory('myFactory', myFactory);
```
Where the last item in the array is the function itself, and all non-last values in the array are treated as names of arguments to inject.

This usage in particular is prevalent throughout signalview.

## Tips & Tricks

### Creating a path when using $injector directly
When calling injector directly, you can pass a caller name, to build a chain to be displayed when there's an error.
```javascript
$injector.get('doesntExistService', 'myFactory');
// Unknown provider: doesntExistServiceProvider <- myFactory

$injector.get('doesntExistService');
// Unknown provider: doesntExistServiceProvider
```

Also it chains, so let's say:
```javascript
angular.module().factory('anotherFactory', () => {
  const service = $injector.get('doesntExistService', 'anotherFactory');
  // omitted
});

angular.module().controller('MyController', () => {});
// when instantiated will throw:
// Unknown provider: doesntExistServiceProvider <- myFactoryProvider <- MyController
```

It might be useful, because you can't easily breakpoint into when this error is thrown, and even if you manage to do that, the function calls stack is quite convoluted.

### Getting an instance of $injector
So let's say you are in DevTools, looking at the code, and you need to get an instance of a service which resides in AngularJS's Dependency Injection system.

```javascript
const service = angular.element(document.body).injector().get('MyService');
```

It is synchronous, so no need for `await`.

If for whatever reason in your app AngularJS isn't mounted at the body, right-click on an element which you know to be managed by AngularJS, Inspect element, and then:
```javascript
const service = angular.element($0).injector().get('MyService');
```
`$0` is a reference to an element selected in Elements tab in DevTools in Chrome. Replace `$0` with whatever is it in your browser developer tools.