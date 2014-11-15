# Registering new users
At this point we have the models and serializers needed to represent users. Now we need to build an authentication system. This involves creating the various views and interfaces for registering, logging in and logging out. We will also touch on an `Authentication` service with AngularJS and a few different controllers.

Because we can't log in users that don't exist, it makes sense to start with registration. 

To register a user, we need an API endpoint that will create the user, an AngularJS service to make an AJAX request to the API and a registration form. Let's make the API endpoint first.

## Making the registration API view
Open `authentication/views.py` and replace it's contents with the following code:

    from django.contrib.auth.models import User
    from rest_framework import generics
    from authentication.serializers import UserSerializer


    class UserCreateView(generics.CreateAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

{x: user_create_view}
Make a view called `UserCreateView` in `authentication/views.py`

Let's step through this snippet line-by-line:

    class UserCreateView(generics.CreateAPIView):

Django REST Framework provides a number of generic API views that save you a lot of time. In this case, we are using `generics.CreateAPIView` which accepts a `POST` request and creates an object.

    queryset = User.objects.all()
    serializer_class = UserSerializer

Here we define the query set and serializer that the view operates on. This is pretty standard for just about any generic view that comes with Django REST Framework. Not much can be explained about this without diving into the Django REST Framework source code (which I suggest you try!).

## Adding an API endpoint
Now that we have created the view, we need to add it to the URLs file. Open `thinkster_django_angular_boilerplate/urls.py` and update it to look like so:

    # .. Imports
    from authentication.views import UserCreateView

    urlpatterns = patterns(
         '',
        # ... URLs
        url('^api/v1/users/$', UserCreateView.as_view(), name='user-create'),

        url(r'^', TemplateView.as_view(template_name='static/index.html')),
    )

{x: url_user_create}
Add an API endpoint for `UserCreateView`

*NOTE: It is very important that the last URL in the above snippet always be the last URL. This is known as a passthrough route. It accepts all requests not matched by any other rules and sends them to the front end for AngularJS to process. The order of other URLs is insignificant.*

## An Angular service for registering new users
With the API endpoint in place, we can create an AngularJS service that will handle communication between the client and the server.

Make a file in `static/javascripts/authentication/services/` called `authentication.service.js` and add the following code:

*NOTE: Feel free to leave the comments out of your own code. It takes a lot to time to type them all out!*

    /**
    * Authentication
    * @namespace thinkster.authentication.services
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.authentication.services')
        .factory('Authentication', Authentication);

      Authentication.$inject = ['$cookies', '$http'];

      /**
      * @namespace Authentication
      * @returns {Factory}
      */
      function Authentication($cookies, $http) {
        /**
        * @name Authentication
        * @desc The Factory to be returned
        */
        var Authentication = {
          register: register
        };

        return Authentication;

        ////////////////////

        /**
        * @name register
        * @desc Try to register a new user
        * @param {string} username The username entered by the user
        * @param {string} password The password entered by the user
        * @param {string} email The email entered by the user
        * @returns {Promise}
        * @memberOf thinkster.authentication.services.Authentication
        */
        function register(username, password, email) {
          return $http.post('/api/v1/users/', {
            username: username,
            password: password,
            email: email
          });
        }
      }
    })();

{x: angularjs_authentication_service}
Make a factory called `Authentication` in `static/javascripts/authentication/services/authentication.service.js`

Let's step through this line-by-line:

    angular
      .module('thinkster.authentication.services')

AngularJS supports the use of modules. Modularization is a great feature because it promotes encapsulation and loose coupling. We make thorough use of Angular's module system throughout the tutorial. For now, all you need to know is that this service is in the `thinkster.authentication.services` module.

    .factory('Authentication', Authentication);

This line registers a factory named `Authentication` on the module from the previous line. 

    function Authentication($cookies, $http) {

Here we define the factory we just registered. We inject the `$cookies` and `$http` services as a dependency. We will be using `$cookies` later.

    var Authentication = {
      register: register
    };

This is personal preference, but I find it's more readable to define your service as a named object and then return it, leaving the details lower in the file. 

    register: function (username, password, email) {

At this point, the `Authentication` service has only one method: `register`, which takes a `username`, `password`, and `email`. We will add more methods to the service as we move forward.
 
    return $http.post('/api/v1/users/', {
      username: username,
      password: password,
      email: email
    });

As mentioned before, we need to make an AJAX request to the API endpoint we made. As data, we include the `username`, `password` and `email` parameters this method received. We have no reason to do anything special with the response, so we will let the caller of `Authentication.register` handle the callback.

## Making an interface for registering new users
Let's begin creating the interface users will use to register. Begin by creating a file in `static/templates/authentication/` called `register.html` with the following content: 

    <div class="row">
      <div class="col-md-4 col-md-offset-4">
        <h1>Register</h1>

        <div class="well">
          <form role="form" ng-submit="vm.register()">
            <div class="form-group">
              <label for="register__username">Username</label>
              <input type="text" class="form-control" id="register__username" ng-model="vm.username" placeholder="ex. john" />
            </div>

            <div class="form-group">
              <label for="register__email">Email</label>
              <input type="email" class="form-control" id="register__email" ng-model="vm.email" placeholder="ex. john@notgoogle.com" />
            </div>

            <div class="form-group">
              <label for="register__password">Password</label>
              <input type="password" class="form-control" id="register__password" ng-model="vm.password" placeholder="ex. thisisnotgoogleplus" />
            </div>

            <div class="form-group">
              <button type="submit" class="btn btn-primary">Submit</button>
            </div>
          </form>
        </div>
      </div>
    </div>

{x: angularjs_register_template}
Create a `register.html` template

We won't go into much detail this time because this is pretty basic HTML. A lot of the classes come from Bootstrap, which is included by the boilerplate project. There are only two lines that we are going to pay attention to:

    <form role="form" ng-submit="vm.register()">

This is the line responsible for calling `$scope.register`, which we set up in our controller. `ng-submit` will call `vm.register` when the form is submitted. If you have used Angular before, you are probably used to using `$scope`. In this tutorial, we choose to avoid using `$scope` where possible in favor of `vm` for ViewModel. See the [Controllers](https://github.com/johnpapa/angularjs-styleguide#controllers) section of John Papa's AngularJS Style Guide for more on this.

    <input type="text" class="form-control" id="register__username" ng-model="vm.username" placeholder="ex. john" />

On each `<input />`, you will see another directive, `ng-model`. `ng-model` is responsible for storing the value of the input on the ViewModel. This is how we get the username, password, and email when `vm.register` is called.

## Controlling the interface with RegisterController
With a service and interface in place, we need a controller to hook the two together. The controller we create, `RegisterController` will allow us to call the `register` method of the `Authentication` service when a user submits the form we've just built.

Create a file in `static/javascripts/authentication/controllers/` called `register.controller.js` and add the following:

    /**
    * Register controller
    * @namespace thinkster.authentication.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.authentication.controllers')
        .controller('RegisterController', RegisterController);

      RegisterController.$inject = ['$location', '$scope', 'Authentication'];

      /**
      * @namespace RegisterController
      */
      function RegisterController($location, $scope, Authentication) {
        var vm = this;

        vm.register = register;

        /**
        * @name register
        * @desc Register a new user
        * @memberOf thinkster.authentication.controllers.RegisterController
        */
        function register() {
          Authentication.register(vm.username, vm.password, vm.email);
        }
      }
    })();


{x: angularjs_register_controller}
Make a controller named `RegisterController` in `static/javascripts/authentication/controllers/register.controller.js`

As usual, we will skip over the familiar and talk about new concepts.

    .controller('RegisterController', RegisterController);

This is similar to the way we registered our service. The difference is that, this time, we are registering a controller.

    vm.register = register;

`vm` allows the template we just created to access the `register` method we define later in the controller.

    Authentication.register(vm..username, vm..password, vm..email);

Here we call the service we created a few minutes ago. We pass in a username, password and email from `vm`. 

## Registration Routes and Modules
Let's set up some client-side routing so users of the app can navigate to the register form.

Create a file in `static/javascripts` called `thinkster.routes.js` and add the following:

    (function () {
      'use strict';

      angular
        .module('thinkster.routes')
        .config(config);

      config.$inject = ['$routeProvider'];

      /**
      * @name config
      * @desc Define valid application routes
      */
      function config($routeProvider) {
        $routeProvider.when('/register', {
          controller: 'RegisterController', 
          controllerAs: 'vm',
          templateUrl: '/static/templates/authentication/register.html'
        }).otherwise('/');
      }
    })();


{x: angularjs_register_route}
Define a route for the registration form

There are a few points we should touch on here.

    .config(config);

Angular, like just about any framework you can imagine, allows you to edit it's configuration. You do this with a `.config` block. 

    function config($routeProvider) {

Here, we are injecting `$routeProvider` as a dependency, which will let us add routing to the client.

    $routeProvider.when('/register', {

`$routeProvider.when` takes two arguments: a path and an options object. Here we use `/register` as the path because thats where we want the registration form to show up.

    controller: 'RegisterController',
    controllerAs: 'vm',

One key you can include in the options object is `controller`. This will map a certain controller to this route. Here we use the `RegisterController` controller we made earlier. `controllerAs` is another option. This is required to use the `vm` variable. In short, we are saying that we want to refer to the controller as `vm` in the template.

    templateUrl: '/static/templates/authentication/register.html'

The other key we will use is `templateUrl`. `templateUrl` takes a string of the URL where the template we want to use for this route can be found.

    }).otherwise('/');

We will add more routes as we move forward, but it's possible a user will enter a URL that we don't support. When this happens, `$routeProvider.otherwise` will redirect the user to the path specified; in this case, '/'.

## Setting up AngularJS modules
Let us quickly discuss modules in AngularJS.

In Angular, you must define modules prior to using them. So far we need to define `thinkster.authentication.services`, `thinkster.authentication.controllers`, and `thinkster.routes`. Because `thinkster.authentication.services` and `thinkster.authentication.controllers` are submodules of `thinkster.authentication`, we need to create a `thinkster.authentication` module as well.

Create a file in `static/javascripts/authentication/` called `authentication.module.js` and add the following:

    (function () {
      'use strict';

      angular
        .module('thinkster.authentication', [
          'thinkster.authentication.controllers',
          'thinkster.authentication.services'
        ]);

      angular
        .module('thinkster.authentication.controllers', []);

      angular
        .module('thinkster.authentication.services', ['ngCookies']);
    })();

{x: angularjs_authentication_module}
Define the `thinkster.authentication` module and it's dependencies

There are a couple of interesting syntaxes to note here.

    angular
      .module('thinkster.authentication', [
        'thinkster.authentication.controllers',
        'thinkster.authentication.services'
      ]);

This syntax defines the module `thinkster.authentication` with `thinkster.authentication.controllers` and `thinkster.authentication.services` as dependencies.

    angular
      .module('thinkster.authentication.controllers', []);

This syntax defines the module `thinkster.authentication.controllers` with no dependencies.

Now we need define to include `thinkster.authentication` and `thinkster.routes` as dependencies of `thinkster`.

Open `static/javascripts/thinkster.js`, define the required modules, and include them as dependencies of the `thinkster` module. Note that `thinkster.routes` relies on `ngRoute`, which is included with the boilerplate project.

    (function () {
      'use strict';

      angular
        .module('thinkster', [
          'thinkster.routes',
          'thinkster.authentication'
        ]);

      angular
        .module('thinkster.routes', ['ngRoute']);
    })();

{x: angularjs_thinkster_module}
Update the `thinkster` module to include it's new dependencies

## Hash routing
By default, Angular uses a feature called hash routing. If you've ever seen a URL that looks like `www.google.com/#/search` then you know what I'm talking about. Again, this is personal preference, but I think those are incredibly ugly. To get rid of hash routing, we can enabled `$locationProvider.html5Mode`. In older browsers that do not support HTML5 routing, Angular will intelligently fall back to hash routing.

Create a file in `static/javascripts/` called `thinkster.config.js` and give it the following content:

    (function () {
      'use strict';

      angular
        .module('thinkster.config')
        .config(config);

      config.$inject = ['$locationProvider'];

      /**
      * @name config
      * @desc Enable HTML5 routing
      */
      function config($locationProvider) {
        $locationProvider.html5Mode(true);
        $locationProvider.hashPrefix('!');
      }
    })();


{x: angularjs_html5mode_config}
Enable HTML5 routing for AngularJS

As mentioned, enabling `$locationProvider.html5Mode` gets rid of the hash sign in the URL. The other setting here, `$locationProvider.hashPrefix`, turns the `#` into a `#!`. This is mostly for the benefit of search engines.

Because we are using a new module here, we need to open up `static/javascripts/thinkster.js`, define the module, and include is as a dependency for the `thinkster` module.

    angular
      .module('thinkster', [
        'thinkster.config',
        // ...
      ]);

    angular
      .module('thinkster.config', []);

{x: angularjs_config_module}
Define the `thinkster.config` module

## Include new .js files
In this chapter so far, we have already created a number of new JavaScript files. We need to include these in the client by adding them to `templates/javascripts.html` inside the `{% compress js %}` block.

Open `templates/javascripts.html` and add the following above the `{% endcompress %}` tag:

    <script type="text/javascript" src="{% static 'javascripts/thinkster.config.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/thinkster.routes.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/authentication.module.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/services/authentication.service.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/controllers/register.controller.js' %}"></script>

{x: django_javascripts}
Add the new JavaScript files to `templates/javascripts.html`

## Handling CSRF protection
Because we are using session-based authentication, we have to worry about CSRF protection. We don't go into detail on CSRF here because it's outside the scope of this tutorial, but suffice it to say that CSRF is very bad.

Django, by default, stores a CSRF token in a cookie named `csrftoken` and expects a header with the name `X-CSRFToken` for any dangerous HTTP request (`POST`, `PUT`, `PATCH`, `DELETE`). We can easily configure Angular to handle this.

Open up `static/javascripts/thinkster.js` and add the following under your module definitions:

    angular
      .module('thinkster')
      .run(run);

    run.$inject = ['$http'];

    /**
    * @name run
    * @desc Update xsrf $http headers to align with Django's defaults
    */
    function run($http) {
      $http.defaults.xsrfHeaderName = 'X-CSRFToken';
      $http.defaults.xsrfCookieName = 'csrftoken';
    }

{x: angularjs_run_csrf}
Configure AngularJS CSRF settings

## Checkpoint
Try registering a new user by running your server (`python manage.py runserver`), visiting `http://localhost:8000/register` in your browser and filling out the form.

If the registration worked, you can view the last `User` object created by opening the shell (`python manage.py shell`) and running the following commands:

    >>> from django.contrib.auth.models import User
    >>> User.objects.latest('date_joined')

The `User` object returned should match the one you just created.

{x: checkpoint_register_user}
Register a new user at `http://localhost:8000/register` and confirm the `User` object was created
