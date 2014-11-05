## Login
Now that we have a semi-working registration system, we need to let users log in. As it turns out, this is part of what we are missing from the registration system. Once a user registers, we should automatically log them in.

To get started, we will create views for logging in and logging out. Once those are done we will progress in a fashion similar to the registration systems: services, controllers, etc.

## Login: API Views and URLs
Open up `authentication/views.py` and add the following imports and class:

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
Create a `LoginView` in `authentication/views.py`

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

## Login: AngularJS Service
Let's add some more methods to our `Authentication` service. We will do this in two stages. First we will add a `login()` method and then we will add some utility methods for storing session data in the browser.

Open `static/javascripts/authentication/services/authentication.service.js` and add the following method to the `Authentication` object we created earlier:

    login: function (username, password) {
      return $http.post('/api/v1/auth/login/', {
        username: username, password: password
      });
    }

{x: angularjs_authentication_service_login}
Add a `login` method to your `Authentication` service

Much like the `register()` method from before, `login()` returns makes an AJAX request to our API and returns a promise.

Now let's talk about a few utility methods we need for managing session information on the client.

We want to display information about the currently authenticated user in the navigation bar at the top of the page. This means we will need a way to store the response returned by `login()`. We will also need a way to retrieve the authenticated user. Finally, we need need a way to unauthenticate the user in the browser (this is different than logging out).

Given these requirements, I suggest three methods: `getAuthenticatedUser`, `setAuthenticatedUser`, and `unauthenticate`.

Let's implement these now. Update `authentication.service.js` like so:

    angular.module('borg.authentication.services')
      .service('Authentication', function ($cookies, $http) {
        var Authentication = {
          // ...
          getAuthenticatedUser: function () {
            if (!$cookies.authenticatedUser) {
              return;
            }

            return JSON.parse($cookies.authenticatedUser);
          },

          setAuthenticatedUser: function (user) {
            $cookies.authenticatedUser = JSON.stringify(user);
          },

          unauthenticate: function () {
            delete $cookies.authenticatedUser;
          }
        };

        return Authentication;
      });

{x: angularjs_authentication_service_utilities}
Add `getAuthenticatedUser`, `setAuthenticatedUser`, and `unauthenticate` methods to your `Authentication` service

Let's step through this code line-by-line.

    .service('Authentication', function ($cookies, $http) {

Notice that we have injected another dependency: the `$cookies` service. The `$cookies` service is provided by `ngCookies`, so we will need to add that as a dependency for the `borg.authentication.services` module.

    if (!$cookies.authenticatedUser) {

We have chosen to store the authenticated user in the cookies for this page. If there is no `authenticatedUser` cookie set, then there is no authenticated user and we can return `undefined`.

    return JSON.parse($cookies.authenticatedUser);

Because JavaScript objects can not be stored in a cookie, we have to store the authenticated user a JSON. We parse this JSON when we retrieve the current user.

    $cookies.authenticatedUser = JSON.stringify(user);

As was just mentioned, the authenticated user must be stored as JSON.

    delete $cookies.authenticatedUser;

Unauthenticating is actually quite simple. All we have to do is remove the `authenticatedUser` cookie.

## Login: AngularJS Controller and Template
We now have `Authentication.login()` to log a user in, so let's create the login form, starting with a controller.

Create a file in `static/javascripts/authentication/controllers/` called `login.controller.js` and add the following contents:

    angular.module('borg.authentication.controllers')
      .controller('LoginController', function ($location, $scope, Authentication) {
        // Logged in users should not be on this page.
        if (Authentication.getAuthenticatedUser()) {
          $location.url('/');
        }

        $scope.login = function () {
          Authentication.login($scope.username, $scope.password)
            .then(function (data, status, headers, config) {
              Authentication.setAuthenticatedUser(data.data);

              window.location = '/';
            }, function (data, status, headers, config) {
              
            });
        };
      });

{x: angularjs_login_controller}
Create a `LoginController` with AngularJS

Time to walk through the code.

    // Logged in users should not be on this page.
    if (Authentication.getAuthenticatedUser()) {
      $location.url('/');
    }

As the comment suggests, if a user is already authenticated, they have no business on the login page. We solve this by redirecting the user to the index page. 

We should do this on the registration page too. When we wrote the registration controller, we didn't have `Authentication.getAuthenticatedUser()`. We will update `RegisterController` shortly.

    Authentication.setAuthenticatedUser(data.data);

If the call to `Authentication.login()` is successful, we store the authenticated user so we have access to it later.

    console.error('Epic fail!');

We will add better error-handling later, but now is not the time.

Let's quickly add a template for the login form before moving on.

Create a file in `static/templates/authentication/` named `login.html` and add the following:

    <div class="row">
      <div class="col-md-4 col-md-offset-4">
        <h1>Login</h1>

        <div class="well">
          <form role="form" ng-submit="login()">
            <div class="form-group">
              <label for="login__username">Username</label>
              <input type="text" class="form-control" id="login__username" ng-model="username" placeholder="ex. hugh" />
            </div>

            <div class="form-group">
              <label for="login__password">Password</label>
              <input type="password" class="form-control" id="login__password" ng-model="password" placeholder="ex. weareborg" />
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

We won't spend any time talking about this because it's no different than the template for the registration form.

## Registration: AngularJS Controller
Taking a step back, let's add a check to `RegisterController` and redirect the user if they are already authenticated.

Open `static/javascripts/authentication/controllers/register.controller.js` and add the following just inside the definition of the controller:

    // Logged in users should not be on this page.
    if (Authentication.getAuthenticatedUser()) {
      $location.url('/');
    }

Don't forget to inject the `$location` service as a dependency of `RegisterController`:

    .controller('RegisterController', function ($location, $scope, Authentication) {

{x: angularjs_register_controller_auth}
Check for authenticated users in `RegisterController`

If you remember, we also talked about logging a user in automatically when they register. Since we are already in `register.controller.js`, let's go ahead and handle that now. Below the check for authenticated users, add the following:

    var login = function (username, password) {
      Authentication.login(username, password).then(
        function (data, status, headers, config) {
          Authentication.setAuthenticatedUser(data.data);

          window.location = '/';
        }, 
        function (data, status, headers, config) {
          console.log('Epic failure!');
        }
      );
    };

{x: angularjs_register_controller_login}
Add a `login()` function to `RegisterController`

Now update `$scope.register` to look like this:

    $scope.register = function () {
      Authentication.register($scope.username, $scope.password, $scope.email).then(
        function (data, status, headers, config) {
          login($scope.username, $scope.password);
        },
        function (data, status, headers, config) {
          console.log('Epic failure!');
        }
      );
    };

{x: angularjs_register_controller_update_register}
Update `$scope.register()`

## Refactoring Controllers
If you are paying attention, then you may notice that `login()` looks a lot like `Authentication.login()`. This means that we need to do some refactoring.

As we aren't doing anything special in the success and error callbacks, we can move those into their respective methods in the `Authentication` service.

Go ahead and update the `login()` and `register()` methods in your `Authentication` service to handle the success and error callbacks like so:

    register: function (username, password, email) {
      return $http.post('/api/v1/users/', {
        username: username,
        password: password,
        email: email
      }).then(function (data, status, headers, config) {
        Authentication.login(username, password);
      }, function(data, status, headers, config) {
        console.error('Epic failure!');
      });
    },

    login: function (username, password) {
      return $http.post('/api/v1/auth/login/', {
        username: username, password: password
      }).then(function (data, status, headers, config) {
        Authentication.setAuthenticatedUser(data.data);

        window.location = '/';
      }, function(data, status, headers, config) {
        console.error('Epic failure!');
      });
    },

{x: angularjs_authentication_service_update_login_register}
Update `login` and `register` in your `Authentication` service

Now that we are handling everything inside `Authentication`, we can simplify out `RegisterController` and `LoginController` controllers:

    angular.module('borg.authentication.controllers')
      .controller('RegisterController', function ($location, $scope, Authentication) {
        // Logged in users should not be on this page.
        if (Authentication.getAuthenticatedUser()) {
          $location.url('/');
        }

        $scope.register = function () {
           Authentication.register($scope.username, $scope.password, $scope.email);
        };
      });

{x: angularjs_register_controller_refactor}
Refactor `RegisterController`

    angular.module('borg.authentication.controllers')
      .controller('LoginController', function ($location, $scope, Authentication) {
        // Logged in users should not be on this page.
        if (Authentication.getAuthenticatedUser()) {
          $location.url('/');
        }

        $scope.login = function () {
          Authentication.login($scope.username, $scope.password);
         };
      });

{x: angularjs_login_controller_refactor}
Refactor `LoginController`

## Login: AngularJS Routes and Modules
The next step is to create the client-side routes for the login form and add define any new modules we've created.

Open up `static/javascripts/borg.routes.js` and add a route for the login form:

    $routeProvider.when('/register', {
      controller: 'RegisterController',
      templateUrl: '/static/templates/authentication/register.html'
    }).when('/login', {
      controller: 'LoginController',
      templateUrl: '/static/templates/authentication/login.html'
    }).otherwise('/');

{x: angularjs_login_route}
Add a route for `LoginController`

*NOTE: See how you can chain calls to `$routeProvider.when()`? Going forward, we will not ignore old routes for brevity. Just keep in mind that these calls should be chained.*

As far as modules are concerned, we haven't actually created any new ones since the last time we talked about it! However, we did start using the `$cookies` service, which requires `ngCookies` be included. 

Update the definition of the `borg.authentication.services` module in `authentication.module.js` to look like so:

    angular.module('borg.authentication.services', ['ngCookies']);

{x: angularjs_authentication_module_ngcookies}
Make `ngCookies` a dependency of the `.borg.authentication.services` module

## Login: Include new .js files
If you can believe it, we've only created one new JavaScript file since the last time: `login.controller.js`. Let's add it to `javascripts.html` with the other JavaScript files:

    <script type="text/javascript" src="{% static 'javascripts/authentication/controllers/login.controller.js' %}"></script>

{x: angularjs_javascripts_login}
Include `login.controller.js` in `javascripts.html`

## Handling CSRF protection
Because we are using session-based authentication, we have to worry about CSRF protection. We don't go into detail on CSRF here because it's outside the scope of this tutorial, but suffice it to say that CSRF is very bad.

Django, by default, stores a CSRF token in a cookie named `csrftoken` and expects a header with the name `X-CSRFToken` for any dangerous HTTP request (`POST`, `PUT`, `PATCH`, `DELETE`). We can easily configure Angular to handle this.

Open up `static/javascripts/borg.js` and add the following to the bottom of the file:

    angular.module('borg')
      .run(function ($http, $cookies) {
        $http.defaults.xsrfHeaderName = 'X-CSRFToken';
        $http.defaults.xsrfCookieName = 'csrftoken';
      });

{x: angularjs_run_csrf}
Configure AngularJS CSRF settings

## Logging in for the first time
Epic! Now you can log into your site! It won't give you any extra functionality for now, but you will be able to see the navigation bar change. Beware though, you can't log out just yet. We will implement the logout functionality next.

Go to the login page, fill out the form when either of the accounts you created earlier, and log in.

Take another breather. You've earned it! When you get back we will jump on implementing a logout feature and then call it a day for authentication.

{x: high_five_2}
Give yourself another high five. Cmon guys, I don't have all night. High five yourself already.


