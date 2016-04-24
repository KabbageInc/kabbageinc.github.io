---
layout: post
title:  "Building Modules with AngularJS"
date:   2015-06-09
author: Joseph Jo
categories: angular
redirect_from:
  - post/121124345117/building-modules-with-angularjs-at-kabbage
---
TL;DR: Angular custom directives are powerful and helped us build modules that can be placed anywhere on any page.

### Intro

Five months ago Kabbage formed a brand new team to create an app that helps our team provide amazing customer service. This app is built on top of the existing Kabbage platform.

A brand new team started the first sprint in earnest in January.

We started with Django as our web framework using the built-in Django templates in the front-end. After a few sprints and seeing the iteration of the interface, we realized that we needed something more robust in the client to deal with complex dashboard states, CRUD actions that don’t require a full page reload, and to provide a seamless user experience.

After some deliberation we settled on AngularJS, a popular client-side MVC framework.

### Setup

We use Django templates for the foundation of all our pages (base) and for primarily static pages (404, 500, auth). We run multiple Angular apps that are isolated from one another on the Django pages to keep the size of our apps down and to reduce complexity.

We use the Django REST Framework to serialize data and to create multiple API endpoints for Angular to call.

In Angular we use UI Router as the routing framework. UI Router brings in the concept of states in addition to routes, allowing for nested states and views.

### Angular modules

In our user account page/Angular app, we have the concept of modules of information.

A few examples:

* customer information module
* credit reports module
* email module
* account flags module

These modules are grouped into separate routes/states.

Let’s look at an example of a route of “profile” that has two modules of “customer information” and “credit reports.”

# Create the module

First, let’s register the module name (don’t forget to also add it as a dependency to the user account app). We’ll add the UI Router dependency as well.

```javascript
angular.module('profile', ['ui.router']);
```

Create the factory

Next, we’ll create the factory that will call the API. We’re using the ngResource module that allows us to easily interact with our REST API.

```javascript
angular.module('profile')
  .factory('UserFactory', ['$resource', function($resource) {
    var userFactory = {};

    userFactory.getUserDetails = function(guid) {
      return $resource('/account/:guid/', {
        guid: guid, format: 'json'
        }, {
          get: {method: 'GET', cache: true}
        }
      );
    };

    userFactory.getCreditReports = function(guid) {
      return $resource(
        '/creditreports/:guid/', {
          guid: guid, format: 'json'
        }, {
          query: {method: 'GET', isArray: true, cache: true}
        }
      );
    };

    return userFactory;
  }]);
```

### Set up the config

Then, we’ll set up the config block, which includes the state/route. This is where the factory is called and we retrieve the data.

```javascript
angular.module('profile')
  .config(['$stateProvider', function($stateProvider) {
    $stateProvider
      .state('profile', {
        controller: 'CustomerInfoCtrl',
        resolve: {
          customerInfo: ['$scope', 'UserFactory', function($scope, UserFactory) {
            return UserFactory.getUserDetails(guid).get().$promise;
          }],
          creditReports: ['$scope', 'UserFactory', function($scope, UserFactory) {
            return UserFactory.getCreditReports($scope.guid).query().$promise;
          }]
        },
        templateUrl: STATIC_URL + 'profile.html',
        url: '/profile'
      });
  }]);
```

The state will block until the promise is resolved. We set the route here to “profile”, and assign it to a controller and a template.

### Template

```html
<div class="page-header">
    <h2>Profile</h2>
</div>

<div class="customer_information module">[...]</div>
<div class="credit_reports module">[...]</div>
```

### The controller

The controller loads in the data from the config’s resolve.

```javascript
angular.module('profile')
  .controller('CustomerInfoCtrl', ['$scope', 'customerInfo', 'creditReports',
    function($scope, customerInfo, creditReports) {
      $scope.account = customerInfo;
      $scope.credit_report = creditReport;
      [...]
  }]);
```

### Trouble in paradise

This paradigm worked well for us: create a state, add a config that calls a factory to get data from the REST API for each module, then pass the data to the controller. For simple apps, this works. However, as our app started to get more complex, states started getting harder to maintain.

Then, last week we ran into a problem that resulted in us taking a second look at our approach.

We had a requirement of a module that would be placed beneath existing modules on any page when clicked in the header. Also, there was the future concept of “favoriting” modules in a quick view route.

This presented a challenge since for any given module the controller, template, and the API call were all dependent on the URL. Along with the static modules that would always be on a particular route, we wanted the ability to add specific modules to whatever page we are on and have modules appear ad hoc. We also wanted to untie modules to a URL even if they would always remain together at that URL.

### A new paradigm

The solution we uncovered was using the power of [custom directives](https://docs.angularjs.org/guide/directive).

Each module would be a separate directive with it’s own service, controller, and template. With this new approach, states are not tied to any specific module and would simply be a name, URL, and a template. The template is basically the header and the directive for any modules indefinite to the page.

### Template

```html
<div class="page-header">
    <h2>Profile</h2>
</div>

<customer-information></customer-information>
<credit-reports></credit-reports>
```

### Config

```javascript
angular.module('profile')
  .config(['$stateProvider', function($stateProvider) {
    $stateProvider
      .state('profile', {
        templateUrl: STATIC_URL + 'profile_state.html',
        url: '/profile'
      });
  }]);
```

For our above example, we’ll also create a new module for credit reports with it’s own factory, controller, and directive. Let’s take a look at just the customerInformation directive for now.

### Factory

The factory remains largely the same except that getCreditReports has been moved into a factory within a newly created creditReports module (not shown here).

```javascript
angular.module('profile')
  .factory('UserFactory', ['$resource', function($resource) {
    var userFactory = {};

    userFactory.getUserDetails = function(guid) {
      return $resource('/account/:guid/', {
        guid: guid, format: 'json'
      }, {
        get: {method: 'GET', cache: true}
      });
    };

    return userFactory;
  }]);
```

### Controller

```javascript
angular.module('profile')
  .controller('CustomerInfoCtrl', ['$scope',
    function($scope) {
      [...]
  }]);
```

### Directive

```javascript
angular.module('profile')
  .directive('customerInformation', ['UserFactory',
    function(UserFactory) {
      return {
        controller: 'CustomerInfoCtrl',
        link: function(scope, element, attrs) {
          UserFactory.getUserDetails(scope.guid).get().$promise;
        },
        restrict: 'EA',
        templateUrl: STATIC_URL + 'customer_information.html'
      };
    }]);
```

### Conclusion

As you see, using directives is an awesome way of separating modules out and creating reusable components.

Instead of using routes and states to structure complex apps, we can use directives, which are location independent and can be placed on any page.

Directives are very powerful and should be used to encapsulate complex widgets and behaviors.
