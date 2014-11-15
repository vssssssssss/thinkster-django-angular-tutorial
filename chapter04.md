# Logging users in
Now that users can register, they need a way to log in. As it turns out, this is part of what we are missing from our registration system. Once a user registers, we should automatically log them in.

To get started, we will create views for logging in and logging out. Once those are done we will progress in a fashion similar to the registration systems: services, controllers, etc.

## Making the login API view
Open up `authentication/views.py` and add the following:

    import json

    from django.contrib.auth import authenticate, login

    from rest_framework import status, views
    from rest_framework.response import Response


    class LoginView(views.APIView):
        def post(self, request, format=None):
            data = json.loads(request.body)

            username = data.get('username', None)
            password = data.get('password', None)

            user = authenticate(username=username, password=password)

            if user is not None:
                if user.is_active:
                    login(request, user)

                    serialized = UserSerializer(user)

                    return Response(serialized.data)
                else:
                    return Response({
                        'error': 'Awkward! Your account has been disabled.'
                    }, status=status.HTTP_401_UNAUTHORIZED)
            else:
                return Response({
                    'error': 'Looks like your username or password is wrong. :('
                }, status=status.HTTP_400_BAD_REQUEST)

{x: django_login_view}
Make a view called `LoginView` in `authentication/views.py`

This is a longer snippet than we've seen in the past, but we will approach it the same way: by talking about what's new and ignoring what we have already encountered.

     class LoginView(views.APIView):

You will notice that we are not using a generic view this time. Because this view does not perfect a generic activity like creating or updating an object, we must start with something more basic. Django REST Framework's `views.APIView` is what we use. While `APIView` does not handle everything for us, it does give us much more than standard Django views do. In particular, `views.APIView` are made specifically to handle AJAX requests. This turns out to save us a lot of time.

    def post(self, request, format=None):

Unlike generic views, we must handle each HTTP verb ourselves. Logging in should typically be a `POST` request, so we override the `self.post()` method.

    user = authenticate(username=username, password=password)

Django provides a nice set of utilities for authenticating users. The `authenticate()` method is the first utility we will cover. `authenticate()` takes a username and a password. Django then checks the database for a `User` with `username` *username*. If one is found, Django will try to verify the given password. If the username and password are correct, the `User` found by `authenticate()` is returned. If either of these steps fail, `authenticate()` will return `None`.

    if user is not None:
        # ...
    else:
        return Response({
            'error': 'Looks like your username or password is wrong. :('
        }, status=status.HTTP_400_BAD_REQUEST)

In the event that `authenticate()` returns `None`, we respond with a `400` status code and tell the user that the username/password combination they provided is invalid.

    if user.is_active:
        # ...
    else:
        return Response({
            'error': 'Awkward! Your account has been disabled.'
        }, status=status.HTTP_401_UNAUTHORIZED)

If the user's account is for some reason inactivate, we respond with a `401` status code. Here we simply say that the account has been disabled.

    login(request, user)

If `authenticate()` success and the user is active, then we use Django's `login()` utility to create a new session for this user.

    serialized = UserSerializer(user)

    return Response(serialized.data)

We want to store some information about this user in the browser if the login request succeeds, so we serialize the `User` object found by `authenticate()` and return the resulting JSON as the response.

## Adding a login API endpoint
Just as we did with the `UserCreateView`, we need to add a route for `LoginView`.

Open up `thinkster_django_angular_boilerplate/settings.py` and add the following route:

    from authentication.views import LoginView, UserCreateView

    urlpatterns = patterns(
        # ...
        url(r'^api/v1/auth/login/$', LoginView.as_view(), name='login'),

        url(r'^', TemplateView.as_view(template_name='static/index.html')),
    )

{x: url_login}
Add an API endpoint for `LoginView`

## Authentication Service
Let's add some more methods to our `Authentication` service. We will do this in two stages. First we will add a `login()` method and then we will add some utility methods for storing session data in the browser.

Open `static/javascripts/authentication/services/authentication.service.js` and add the following method to the `Authentication` object we created earlier:

    /**
     * @name login
     * @desc Try to log in with username `username` and password `password`
     * @param {string} username The username entered by the user
     * @param {string} password The password entered by the user
     * @returns {Promise}
     * @memberOf thinkster.authentication.services.Authentication
     */
    function login(username, password) {
      return $http.post('/api/v1/auth/login/', {
        username: username, password: password
      });
    }

Make sure to expose it as part of the service:

    var Authentication = {
      login: login,
      register: register
    };

{x: angularjs_authentication_service_login}
Add a `login` method to your `Authentication` service

Much like the `register()` method from before, `login()` returns makes an AJAX request to our API and returns a promise.

Now let's talk about a few utility methods we need for managing session information on the client.

We want to display information about the currently authenticated user in the navigation bar at the top of the page. This means we will need a way to store the response returned by `login()`. We will also need a way to retrieve the authenticated user. We need need a way to unauthenticate the user in the browser. Finally, it would be nice to have an easy way to check if the current user is authenticated.

*NOTE: Unauthenticating is different from logging out. When a user logs out, we need a way to remove all remaining session data from the client.*

Given these requirements, I suggest three methods: `getAuthenticatedUser`, `isAuthenticated`, `setAuthenticatedUser`, and `unauthenticate`.

Let's implement these now. Add each of the following functions to the `Authentication` service:

    /**
     * @name getAuthenticatedUser
     * @desc Return the currently authenticated user
     * @returns {object|undefined} User if authenticated, else `undefined`
     * @memberOf thinkster.authentication.services.Authentication
     */
    function getAuthenticatedUser() {
      if (!$cookies.authenticatedUser) {
        return;
      }

      return JSON.parse($cookies.authenticatedUser);
    }

If there is no `authenticatedUser` cookie (set in `setAuthenticatedUser()`), then return; otherwise return the parsed user object from the cookie.

    /**
     * @name isAuthenticated
     * @desc Check if the current user is authenticated
     * @returns {boolean} True is user is authenticated, else false.
     * @memberOf thinkster.authentication.services.Authentication
     */
    function isAuthenticated() {
      return !!$cookies.authenticatedUser;
    }

Return the boolean value of the `authenticatedUser` cookie. 

    /**
     * @name setAuthenticatedUser
     * @desc Stringify the user object and store it in a cookie
     * @param {Object} user The user object to be stored
     * @returns {undefined}
     * @memberOf thinkster.authentication.services.Authentication
     */
    function setAuthenticatedUser(user) {
      $cookies.authenticatedUser = JSON.stringify(user);
    }

Set the `authenticatedUser` cookie to a stringified version of the `user` object.

    /**
     * @name unauthenticate
     * @desc Delete the cookie where the user object is stored
     * @returns {undefined}
     * @memberOf thinkster.authentication.services.Authentication
     */
    function unauthenticate() {
      delete $cookies.authenticatedUser;
    }

Remove the `authenticatedUser` cookie.

Again, don't forget to expose these methods as part of the service:

    var Authentication = {
      getAuthenticatedUser: getAuthenticatedUser,
      isAuthenticated: isAuthenticated,
      login: login,
      register: register,
      setAuthenticatedUser: setAuthenticatedUser,
      unauthenticate: unauthenticate
    };


{x: angularjs_authentication_service_utilities}
Add `getAuthenticatedUser`, `isAuthenticated`, `setAuthenticatedUser`, and `unauthenticate` methods to your `Authentication` service

Before we move on to the login interface, let's quickly update the `login` method of the `Authentication` service to use one of these new utility methods. Replace `Authentication.login` with the following:

    /**
     * @name login
     * @desc Try to log in with username `username` and password `password`
     * @param {string} username The username entered by the user
     * @param {string} password The password entered by the user
     * @returns {Promise}
     * @memberOf thinkster.authentication.services.Authentication
     */
    function login(username, password) {
      return $http.post('/api/v1/auth/login/', {
        username: username, password: password
      }).then(loginSuccessFn, loginErrorFn);

      /**
       * @name loginSuccessFn
       * @desc Set the authenticated user and redirect to index
       */
      function loginSuccessFn(data, status, headers, config) {
        Authentication.setAuthenticatedUser(data.data);

        window.location = '/';
      }

      /**
       * @name loginErrorFn
       * @desc Log "Epic failure!" to the console
       */
      function loginErrorFn(data, status, headers, config) {
        console.error('Epic failure!');
      }
    }

{x: update_authentication_login}
Update `Authentication.login` to use our new utility methods

## Making a login interface
We now have `Authentication.login()` to log a user in, so let's create the login form. Open up `static/templates/authentication/login.html` and add the following HTML:

    <div class="row">
      <div class="col-md-4 col-md-offset-4">
        <h1>Login</h1>

        <div class="well">
          <form role="form" ng-submit="vm.login()">
            <div class="alert alert-danger" ng-show="error" ng-bind="error"></div>

            <div class="form-group">
              <label for="login__username">Username</label>
              <input type="text" class="form-control" id="login__username" ng-model="vm.username" placeholder="ex. john" />
            </div>

            <div class="form-group">
              <label for="login__password">Password</label>
              <input type="password" class="form-control" id="login__password" ng-model="vm.password" placeholder="ex. thisisnotgoogleplus" />
            </div>

            <div class="form-group">
              <button type="submit" class="btn btn-primary">Submit</button>
            </div>
          </form>
        </div>
      </div>
    </div>

{x: angularjs_login_template}
Create a `login.html` template

## Controlling the login interface with LoginController
Create a file in `static/javascripts/authentication/controllers/` called `login.controller.js` and add the following contents:

    /**
    * LoginController
    * @namespace thinkster.authentication.controllers
    */
    (function () {
      'use static';

      angular
        .module('thinkster.authentication.controllers')
        .controller('LoginController', LoginController);

      LoginController.$inject = ['$location', '$scope', 'Authentication'];

      /**
      * @namespace LoginController
      */
      function LoginController($location, $scope, Authentication) {
        var vm = this;

        vm.login = login;

        activate();

        /**
        * @name activate
        * @desc Actions to be performed when this controller is instantiated
        * @memberOf thinkster.authentication.controllers.LoginController
        */
        function activate() {
          // If the user is authenticated, they should not be here.
          if (Authentication.isAuthenticated()) {
            $location.url('/');
          }
        }

        /**
        * @name login
        * @desc Log the user in
        * @memberOf thinkster.authentication.controllers.LoginController
        */
        function login() {
          Authentication.login(vm.username, vm.password);
        }
      }
    })();

{x: angularjs_login_controller}
Make a controller called `LoginController` in `static/javascripts/authentication/controllers/login.controller.js`

Let's look at the `activate` function.

    function activate() {
      // If the user is authenticated, they should not be here.
      if (Authentication.isAuthenticated()) {
        $location.url('/');
      }
    }

You will start to notice that we use a function called `activate` a lot throughout this tutorial. There is nothing inherently special about this name; we chose a standard name for the function that will be run when any given controller is instantiated.

As the comment suggests, if a user is already authenticated, they have no business on the login page. We solve this by redirecting the user to the index page. 

We should do this on the registration page too. When we wrote the registration controller, we didn't have `Authentication.isAuthenticated()`. We will update `RegisterController` shortly.

## Back to RegisterController
Taking a step back, let's add a check to `RegisterController` and redirect the user if they are already authenticated.

Open `static/javascripts/authentication/controllers/register.controller.js` and add the following just inside the definition of the controller:

    /**
     * @name activate
     * @desc Actions to be performed when this controller is instantiated
     * @memberOf thinkster.authentication.controllers.RegisterController
     */
    function activate() {
      // If the user is authenticated, they should not be here.
      if (Authentication.isAuthenticated()) {
        $location.url('/');
      }
    }

{x: angularjs_register_controller_auth}
Redirect authenticated users to the index view in `RegisterController`

If you remember, we also talked about logging a user in automatically when they register. Since we are already updating registration related content, let's update the `register` method in the `Authentication` service.

Replace `Authentication.register` when the following:

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
      }).then(registerSuccessFn, registerErrorFn);

      /**
      * @name registerSuccessFn
      * @desc Log the new user in
      */
      function registerSuccessFn(data, status, headers, config) {
        Authentication.login(username, password);
      }

      /**
      * @name registerErrorFn
      * @desc Log "Epic failure!" to the console
      */
      function registerErrorFn(data, status, headers, config) {
        console.error('Epic failure!');
      }
    }

{x: angularjs_register_controller_login}
Update `Authentication.register`

## Making a route for the login interface
The next step is to create the client-side route for the login form.

Open up `static/javascripts/thinkster.routes.js` and add a route for the login form:

    $routeProvider.when('/register', {
      controller: 'RegisterController', 
      controllerAs: 'vm',
      templateUrl: '/static/templates/authentication/register.html'
    }).when('/login', {
      controller: 'LoginController',
      controllerAs: 'vm',
      templateUrl: '/static/templates/authentication/login.html'
    }).otherwise('/');

{x: angularjs_login_route}
Add a route for `LoginController`

*NOTE: See how you can chain calls to `$routeProvider.when()`? Going forward, we will ignore old routes for brevity. Just keep in mind that these calls should be chained and that the first route matched will take control.*

## Include new .js files
If you can believe it, we've only created one new JavaScript file since the last time: `login.controller.js`. Let's add it to `javascripts.html` with the other JavaScript files:

    <script type="text/javascript" src="{% static 'javascripts/authentication/controllers/login.controller.js' %}"></script>

{x: angularjs_javascripts_login}
Include `login.controller.js` in `javascripts.html`

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
