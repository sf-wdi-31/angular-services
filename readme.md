<!--
Creator: Team, most recent editing by Brianna
Location: SF
-->

![](https://ga-dash.s3.amazonaws.com/production/assets/logo-9f88ae6c9c3871690e33280fcf557f33.png)

# Angular Services

### Why is this important?
<!-- framing the "why" in big-picture/real world examples -->
*This workshop is important because:*

Angular services  are used to organize and share code across an app. They're important for maintaining separation of concerns in an Angular app. "Services" in general are also a pattern across frameworks.

Angular has many built-in services, but it's very common for developers to create their own.

### What are the objectives?
<!-- specific/measurable goal for students to achieve -->
*After this workshop, developers will be able to:*

- Explain motivations for using services.
- Create a custom service.
- Use promises in a custom service.

### Where should we be now?
<!-- call out the skills that are prerequisites -->
*Before this workshop, developers should already be able to:*

- Make `$http` calls to an external API from within a controller.
- Update the view with data from the `$http` calls.


### Controllers Review

> **Question: What is the purpose of a controller?**
>
> Answer: to control the the version of the data that ties into your page.  A controller manages a page's presentation logic. This is why controllers are sometimes though of as [view models](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel).


```js
BooksIndexController.$inject=['$http'];
function BooksIndexController($http) {
  var vm = this;

  $http({
    method: 'GET',
    url: 'https://super-crud.herokuapp.com/books'
  }).then(onBooksIndexSuccess, onError)

  function onBooksIndexSuccess(response){
    console.log('here\'s the get all books response data', response.data);
    vm.books = response.data.books;
  }
  function onError(error){
    console.log('there was an error: ', error);
  }
};
```

> **Question: Then why does the controller have to worry so much about getting the data with `$http`?**
>
> Answer: it shouldn't.


### Single Responsibility & Separation of Concerns

It's time that we let our controller manage just its single responsibility.  Since data management is a separate concern, we'll use a **Service** for that.

Services will:

  * represent a record or set of records.  
  * manage fetching records as needed.

Controllers should not:

  * know where data comes from or how to get it - only who (which service) to ask.
  * work directly with `$http` or any other sort of retrieval protocol.

This is a big change, but it's a best practice.

### Service Syntax

1. To get started with a service, add it to your angular module. You can do this in `app.js` or  - better practice! - in a separate service file.

  ```js
  angular.module('storeApp')
    .service('ReceiptService', ReceiptService);
  ```

1. The service should be a function that has its dependencies injected as usual. Methods that should be available outside the service should be added to the service object (`this`).

  ```js
  ReceiptService.$inject = [];
  function ReceiptService() {
    console.log('inside Receipt service');
    var self = this;  // similar to vm = this
    self.items = [];
    self.queryAllItems = function(){
      return self.items;
    }
    // ... functions to add and remove items... then other utility functions:
    self.calculateTotal = function(){
      return self.items.reduce(function(previousItem, currentItem){
        return previousItem.price + currentItem.price;
      });
    };
    self.discountItem = function(item, percentOff){
      var discountMultiplier = (100-percentOff)/100;
      item.price = item.price*discountMultiplier;
    }
  }
  ```

1. Now the service is ready to be used elsewhere! Here's how we'd make those functions and data available in a controller:

  ```js
  ReceiptController.$inject=['ReceiptService'];
  function ReceiptController( ReceiptService) {
    var vm = this;
    vm.items = ReceiptService.queryAllItems();
    vm.total = ReceiptService.calculateTotal();
    // ...
  }
  ```

**Note**: You might also see an alternative "factory" syntax for creating services. This syntax just sets up a "factory" function to create a service instance.

<details><summary>click for factory syntax example</summary>
```js
angular.module('storeApp')
  .factory('ReceiptService', ReceiptService);

ReceiptService.$inject = [];
function ReceiptService() {
  var shinyNewServiceInstance = {};
  // factory function body that constructs shinyNewServiceInstance
  shinyNewServiceInstance.items = [];
  shinyNewServiceInstance.queryAllItems = function(){
    return self.items;
  }
  // ... and so on!
  return shinyNewServiceInstance;
});
```

The controller side looks the same:

```js
ReceiptController.$inject=['ReceiptService'];
function ReceiptController( ReceiptService ) {
  var vm = this;
  vm.items = ReceiptService.queryAllItems();
  vm.total = ReceiptService.calculateTotal();
  // ...
}
```
</details>

### Refactor Plan: Promises & Service

Remember this promise structure?

```js
function task(input){ // set up function to use with promises
  var deferred = $q.defer();  // create a new 'deferred'

  // write code to **resolve** the promise when appropriate...
  if (input){
    deferred.resolve(data);
  }
  // ... and to **reject** it when appropriate
  else {
    deferred.reject("Failed: ", error);
  }

  // set up access to this eventual (promised) result
  var promise = deferred.promise;  
  return promise;  
}
```

```js
// elsewhere in code,  attach functions saying what to do next
promise.then(successFunction, errorFunction);
```

Our service will abstract the `$http` logic out of the controllers. To do this, the service will build up a set of promises. The controllers will only have to use these promises.



We already know how to write the code we need to implement a service that handles all of our RESTful routes.  The challenge is putting it together.  We'll set up the deferred objects and promises in the service and resolve each one after the appropriate `$http` call completes. Over in the controller, we'll use `.then` to say what should happen when each of the deferred tasks is done.

We're not letting user create books, so we'll use 5 basic methods:

* `query`  - gets all of a resource
* `get`    - gets a specific resource (usually by id)
* `remove` - removes a resource
* `update` - alters an existing resource
* `save`   - store a new instance of a resource

Each of these methods will follow the same basic pattern -- returning a *promise* to whatever controllers use it.


### Sample Code
Let's look at an example.

```js
BookService.$inject = ['$http', '$q'];
function BookService(   $http,   $q ) {
  console.log('inside book service');
  var self = this;  // why not use vm?
  self.book = {};
  self.get = get;     // get one book

  function get(bookId) {
    var deferred = $q.defer();

    $http({
      method: 'GET',
      url: 'https://super-crud.herokuapp.com/books/'+bookId
    }).then(onBookShowSuccess, onError);

    return deferred.promise;

    // note how these functions are defined within the body of another function?
    // that gives them access to variables from that function
    // - see lexical scope & closures https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures

    // how are these function definitions working, when they come after the return?
    function onBookShowSuccess(response) {
      console.log('BookService: book', bookId, ':', response.data);
      self.book = response.data;
      // time to resolve the deferred and choose what to send to the controller
      deferred.resolve(self.book);
    }
    function onError(error){
      console.log('there was an error: ', error);
      self.book = {error: error};
      // server error - reject the deferred and choose what to send to the controller
      deferred.reject(self.book);
    }
  }
}
```

### `ngResource`

Many sites employ RESTful conventions, so they serve things in very similar ways. What if we could reuse a service like this for any API with RESTful routes?

It turns out the Angular community thought of that a long time ago.  The module `ngResource` implements a service very similar to the one we just built! If you're using a RESTful API, you can often replace `$http` with `ngResource`.

<details><summary>**Click for information on using `ngResource`**</summary>

1. Add `ngResource` with a script tag:
	`<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.4.8/angular-resource.js"></script>`

1. Include `ngResource` as a dependency of your angular module.
	`angular.module('libraryApp', ['ngRoute', 'ngResource'])`

1. List `ngResource`'s  `$resource` as a dependency for your Service.

1. `ngResource` will handle most of the data grabbing we'd want, given an API with RESTful routes. See an example service using `ngResource` below:

```js

angular.module('libraryApp')
  .service('BookService', BookService);

BookService.$inject = ['$http', '$q', '$resource'];
function BookService($http, $q, $resource) {
  console.log('service');
  var self = this;  
  self.book = {};  
  self.books = [];  
  resource = $resource('https://super-crud.herokuapp.com/books/:id', { id: '@_id' }, {
      update: {
        method: 'PUT' // this method issues a PUT request
      },
      query: {
        isArray: true,
        transformResponse: function(data) {
            return angular.fromJson(data).books; // grab books array from response data: `{books: [...]}`
        }
      }
    });
  return resource;
}
```
</details>

### Closing Thoughts

- Explain motivations for using services.
- What are three steps you would take to create a custom service?
- Why did we use promises in our custom book service?

