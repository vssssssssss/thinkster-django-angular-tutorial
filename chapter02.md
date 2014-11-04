# Building an authentication system
In the last chapter, we create a `UserProfile` model so we can store some extra information about our users. 

Now we will build an authentication system that let's users register, log in, and log out. We will cover write API views with Django REST Framework, creating an `Authentication` service with AngularJS and finally the templates and controllers that will display various forms to the client.

## Registration
Because we can't log in users that don't exist, it makes sense to start with registration. 

To register a user, we need an API endpoint that will create the user, an AngularJS service to make an AJAX request to the API and a registration form. Let's make the API endpoint first.

## Registration: API Views and URLs
Open `authentication/views.py` and replace it's contents with the following code:

    from django.contrib.auth.models import User
    from rest_frameworks import generics
    from authentication.serializers import UserSerializer


    class UserCreateView(generics.CreateAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

{x: user_create_view}
Create a `UserCreateView` in `authentication/views.py`

As before, we will step through this -- admittedly short -- code snippet.

    class UserCreateView(generics.CreateAPIView):

Django REST Framework provides a number of generic API views that save you a lot of time. In this case, we are using `generics.CreateAPIView` which accepts a `POST` request and creates an object.

    queryset = User.objects.all()
    serializer_class = UserSerializer

Here we define the query set and serializer that the view operates on. This is pretty standard for just about any generic view that comes with Django REST Framework. Not much can be explained about this without diving into the Django REST Framework source code (which I suggest you try!).

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

## Registration: AngularJS Service
With the API endpoint in place, we can create an AngularJS service that will handle communication between the client and the server.

Make a file in `static/javascripts/authentication/services/` called `authentication.service.js` and add the following code:

    angular.module('borg.authentication.services')
      .service('Authentication', function ($http) {
        var Authentication = {
          register: function (username, password, email) {
            return $http.post('/api/v1/users/', {
              username: username,
              password: password,
              email: email
            });
          }
        };

        return Authentication;
      });

{x: angularjs_authentication_service}
Create an `Authentication` service with AngularJS

Let's step through this line-by-line:

       angular.module('borg.authentication.services')

AngularJS supports the use of modules. Modularization is a great feature because it promotes encapsulation and loose coupling. We make thorough use of Angular's module system throughout the tutorial. For now, all you need to know is that this service is in the `borg.authentication.services` module.

    .service('Authentication', function ($http) {

This line defines a service object named `Authentication` on the module from the previous line. We inject the `$http` service as a dependency.

    var Authentication = {

This is personal preference, but I find it's more readable to define your service as a named object and then return it at the end of the file. Returning an anonymous object is another option.

    register: function (username, password, email) {

At this point, the `Authentication` service has only one method: `register`, which takes a `username`, `password`, and `email`. We will add more methods to the service as we move forward.
 
    return $http.post('/api/v1/users/', {
      username: username,
      password: password,
      email: email
    });

As mentioned before, we need to make an AJAX request to the API endpoint we made. As data, we include the `username`, `password` and `email` parameters this method received. We have no reason to do anything special with the response, so we will let the caller of `Authentication.register` handle the callback.

## Registration: AngularJS Controller and Template
We just finished the first iteration of our first service. Now we need something that can call `Authentication.register`. This is where controllers come in. In this section we will create our first AngularJS controller and a template that will being viewable in the client.

Create a file in `static/javascripts/authentication/controllers/` called `register.controller.js` and add the following:

    angular.module('borg.authentication.controllers')
      .controller('RegisterController', function ($scope, Authentication) {
        $scope.register = function () {
          Authentication.register($scope.username, $scope.password, $scope.email).then(
            function (data, status, headers, config) {
              console.log('Success!');
            },
            function (data, status, headers, config) {
              console.error('Epic failure!');
            }
          );
        };
      });

{x: angularjs_register_controller}
Create a `RegisterController` controller with AngularJS

As usual, we will skip over the familiar and talk about new concepts.

    .controller('RegisterController', function ($scope, Authentication) {

This is similar to the way we defined our service. The difference is that this time we are defining a controller object.

    $scope.register = function () {

`$scope` allows the template we will create shortly to access our variables. Here, we are saying the `register` function on `$scope` because we want to call `register()` when our user submits the registration form.

    Authentication.register($scope.username, $scope.password, $scope.email).then(

Here we call the service we created a few minutes ago. We pass in a username, password and email from `$scope`, which we will cover shortly. We then call the `.then()` method because `Authentication.register()` returns a promise.

    function (data, status, headers, config) {
      console.log('Success!');
    },
    function (data, status, headers, config) {
      console.error('Epic failure!');
    }

These are the success and error callbacks for `.then()`. We will fill them in later after we add some more methods to our `Authentication` service.

For now, let's focus on the registration template.

Create a file in `static/templates/authentication/` called `register.html` and add the following:

    <div class="row">
      <div class="col-md-4 col-md-offset-4">
        <h1>Register</h1>

        <div class="well">
          <form role="form" ng-submit="register()">
            <div class="form-group">
              <label for="register__username">Username</label>
              <input type="text" class="form-control" id="register__username" ng-model="username" placeholder="ex. hugh" />
            </div>

            <div class="form-group">
              <label for="register__email">Email</label>
              <input type="email" class="form-control" id="register__email" ng-model="email" placeholder="ex. hugh@borgcollective.org" />
            </div>

            <div class="form-group">
              <label for="register__password">Password</label>
              <input type="password" class="form-control" id="register__password" ng-model="password" placeholder="ex. weareborg" />
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

    <form role="form" ng-submit="register()">

This is the line responsible for calling `$scope.register`, which we set up in our controller. `ng-submit` is what's known as a directive in AngularJS. We will touch on directives later when we create our own, but for now all you need to know is that `ng-submit` will call `$scope.register` when the form is submitted.

    <input type="text" class="form-control" id="register__username" ng-model="username" placeholder="ex. hugh" />

On each `<input />`, you will see another directive, `ng-model`. `ng-model` is responsible for storing the value of the input on `$scope`. `ng-mdodel="username"` means AngularJS should store the value of this input at `$scope.username`. This is how we get the username, password, and email when `$scope.register` is called.

## Registration: AngularJS Routes and Modules
Let's set up some client-side routing so users of the app navigate to the register form.

Create a file in `static/javascripts` called `borg.routes.js` and add the following:

    angular.module('borg.routes')
      .config(function ($routeProvider) {
        $routeProvider.when('/register', {
          controller: 'RegisterController',
          templateUrl: '/static/templates/authentication/register.html'
        }).otherwise('/');
      });

{x: angularjs_register_route}
Define a route for the registration form

There are a few points we should touch on here.

    .config(function ($routeProvider) {

Angular, like just about any framework you can imagine, allows you to set different configurations. You do this with a `.config` block. Here, we are injecting `$routeProvider` as a dependency, which will let us add routing to the client.

    $routeProvider.when('/register', {

`$routeProvider.when` takes two arguments: a path and an options object. Here we use `/register` as the path because thats where we want the registration form to show up.

    controller: 'RegisterController',

One key you can include in the options object is `controller`. This will map a certain controller to this route. Here we use the `RegisterController` controller we made earlier.

    templateUrl: '/static/templates/authentication/register.html'

The other key we will use is `templateUrl`. `templateUrl` takes a string of the URL where the template we want to use for this route can be found.

    }).otherwise('/');

We will add more routes as we move forward, but it's possible a user will enter a URL that we don't support. When this happens, `$routeProvider.otherwise` will redirect the user to the path specified; in this case, '/'.

With all this done, we can go back to our discussion on modules.

In Angular, you must define modules prior to using them. So far we need to define `borg.authentication.services`, `borg.authentication.controllers`, and `borg.routes`. Because `borg.authentication.services` and `borg.authentication.controllers` are submodules of `borg.authentication`, we need to create a `borg.authentication` module as well.

Create a file in `static/javascripts/authentication/` called `authentication.module.js` and add the following:

    angular.module('borg.authentication', [
      'borg.authentication.controllers',
      'borg.authentication.services'
    ]);

    angular.module('borg.authentication.controllers', []);
    angular.module('borg.authentication.services', []);

{x: angularjs_authentication_module}
Define the `borg.authentication` module and it's dependencies

There are a couple of interesting syntaxes to note here.

    angular.module('borg.authentication', [
      'borg.authentication.controllers',
      'borg.authentication.services'
    ]);

This syntax defines the module `borg.authentication` with `borg.authentication.controllers` and `borg.authentication.services` as dependencies.

     angular.module('borg.authentication.controllers', []);

This syntax defines the module `borg.authentication.controllers` with no dependencies.

Now we need define to include `borg.authentication` and `borg.routes` as dependencies of `borg`.

Open `static/javascripts/borg.js`, define the required modules, and include them as dependencies of the `borg` module. Note that `borg.routes` relies on `ngRoute`, which is included with the boilerplate project.

    angular.module('borg', [
        'borg.routes',
        'borg.authentication'
    ]);

    angular.module('borg.routes', ['ngRoute']);

{x: angularjs_borg_module}
Update the `borg` module to include it's new dependencies

## Registration: Hash routing
By default, Angular using a feature called hash routing. If you've ever seen a URL that looks like `www.google.com/#/search` then you know what I'm talking about. Again, this is personal preference, but I think those are incredibly ugly. To get rid of hash routing, we can enabled `$locationProvider.html5Mode`. In older browsers that do not support HTML5 routing, Angular will intelligently fall back to hash routing.

Create a file in `static/javascripts/` called `borg.config.js` and give it the following content:

    angular.module('borg.config')
      .config(function ($locationProvider) {
        $locationProvider.html5Mode(true);
        $locationProvider.hashPrefix('!');
      });

{x: angularjs_html5mode_config}
Enable HTML5 routing for AngularJS

As mentioned, enabling `$locationProvider.html5Mode` gets rid of the hash sign in the URL. The other setting here, `$locationProvider.hashPrefix` turns the `#` into a `#!`. This is mostly for the benefit of search engines.

Because we are using a new module here, we need to open up `static/javascripts/borg.js`, define the module, and include is as a dependency for the `borg` module.

    angular.module('borg', [
      'borg.config',
      // ...
    ]);

    angular.module('borg.config', []);
    // ...

{x: angularjs_config_module}
Define the `borg.config` module

## Registration: Include new .js files
In this chapter so far, we have already created a number of new JavaScript files. We need to include these in the client by adding them to `templates/javascripts.html` inside the `{% compress js %}` block (more on django-compressor later).

Open `templates/javascripts.html` and add the following above the `{% endcompress %}` tag:

    <script type="text/javascript" src="{% static 'javascripts/borg.config.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/borg.routes.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/authentication.module.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/services/authentication.service.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/controllers/register.controller.js' %}"></script>

{x: django_javascripts}
Add the new JavaScript files to `templates/javascripts.html`

## Let's register a user!
At this point we have enough to register a new account. Yay! 

Keeping in mind that we still need to implement the success and error callbacks of `Authentication.register`, let's go ahead and register a user.

Run `python manage.py runserver` from the root directory of your project. Navigate to `http://localhost:8000/` and click on the **Register** button in the top-right corner. Before you submit the form, make sure your browser console is open. Fill in each field and submit the form. If all went well, you should see *Success!* show up in your browser console.

Congratulations! You're well on your way to having a fully working web application built with Django and AngularJS. But we aren't out of the woods just yet! Let's keep on trucking'.

{x: high_five_1}
Give yourself a high five. Seriously. Do it now.

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

## Logout: API Views and URLs
Let's implement the last authentication-related API view.

Open up `authentication/views.py` and add the following imports and class:

    from django.contrib.auth import logout

    from rest_framework import permissions

    class LogoutView(views.APIView):
         permission_classes = (permissions.IsAuthenticated,)

        def post(self, request, format=None):
            logout(request)

            return Response()

{x: django_logout_view}
Add a `LogoutView` to `authentication/views.py`

There are only a few new things to talk about this time.

    permission_classes = (permissions.IsAuthenticated,)

Only authenticated users should be able to hit this endpoint. Django REST Framework's `permissions.IsAuthenticated` handles this for us. If you user is not authenticated, they will get a `403` error.

    logout(request)

If the user is authenticated, all we need to do is call Django's `logout()` method.

    return Response()

There isn't anything reasonable to return when logging out, so we just return an empty response with a `200` status code.

Moving on to the URLs.

Open up `thinkster_django_angular_boilerplate/urls.py` again and add the following import and URL:

    from authentication.views import LogoutView

    urlpatterns = patterns(
        # ...
        url(r'^api/v1/auth/logout/$', LogoutView.as_view(), name='logout'),
        #...
    )

{x: django_url_logout}
Create an API endpoint for `LogoutView`

## Logout: AngularJS Service
The final method you need to add to your `Authentication` service is the `logout()` method.

Add the following method to the `Authentication` service in `authentication.service.js`:

    logout: function (username, password) {
      return $http.post('/api/v1/auth/logout/')
        .then(function (data, status, headers, config) {
          Authentication.unauthenticate();
          
          window.location = '/';
        }, function (data, status, headers, config) {
          console.error('Epic failure!');
        });
      },

{x: angularjs_authentication_service_logout}
Add a `logout()` method to your `Authentication` service

## Logout: AngularJS Controller and Template
There will not actually be a `LogoutController` or `logout.html`. Instead, the navigation bar already contains a logout link for authenticated users. We will create a `NavbarController` for handling the logout buttons `onclick` functionality and we will update the link itself with an `ng-click` attribute.

Create a file in `static/javascripts/static/controllers/` called `navbar.controller.js` and add the following to it:

    angular.module('borg.static.controllers')
      .controller('NavbarController', function ($scope, Authentication) {
        $scope.logout = function () {
          Authentication.logout();
        };
      });

{x: angularjs_navbar_controller}
Create a `NavbarController` in `static/javascripts/static/controllers/navbar.controller.js`

Open `templates/navbar.html` and add an `mg-controller` directive with the value `NavbarController` to the `<nav />` tag like so:

    <nav class="navbar navbar-default" role="navigation" ng-controller="NavbarController">

While you have `templates/navbar.html` open, go ahead and find the logout link and add `ng-click="logout()"` to it like so:

    <li><a href="javascript:void(0)" ng-click="logout()">Logout</a></li>

{x: angularjs_navbar_template_update}
Update `navbar.html` to include the `ng-controller` and `ng-click` directives where appropriate

## Logout: AngularJS Modules
We need to add a few new modules this time around.

Create a file in `static/javascripts/static/` called `static.module.js` and give it the following contents:

    angular.module('borg.static', [
      'borg.static.controllers'
    ]);

    angular.module('borg.static.controllers', []);

And don't forget to update `static/javascripts/borg.js` also:

    angular.module('borg', [
      // ...
      'borg.static'
    ]);

{x: angularjs_static_module}
Define new `borg.static` and `borg.static.controllers` modules

## Logout: Include new .js files
This time around there are a couple new JavaScript files to include. Open up `javascripts.html` and add the following:

    <script type="text/javascript" src="{% static 'javascripts/static/static.module.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/static/controllers/navbar.controller.js' %}"></script>

{x: include_javascript_static}
Include new JavaScript files in `javascripts.html`

## Seriously, go away. You need a break. I know I do.
This has been a long chapter, but we have covered so much ground! Authentication is a very time consuming endeavor, and we haven't even scratched the surface of it. There are many speed optimizations and security concerns to worry about that we haven't identified here.

With that said, you should be incredibly proud of what you've done so far. Most people will never implement an authentication system.

To recap, in this chapter we have touched on registration, login, and logout. You implemented API views, services, and controllers for each feature. The code you've written is both clean and modular and you are now ready to move on to building the rest of your application.

In the next chapter we will begin modeling Thoughts, which are our rendition of typical social network posts/statuses/tweets/whatever. Once we have our models fleshed out, we will move on to serialization and finally move on to creating some more API views. 

A lot of what we cover in the next chapter will be review for you, but repetition is a great way to learn. Because we won't be presenting many new concepts as we go forward, I will leave you with a challenge: From now on, try figuring out what the code will look like before reading the snippets. This turns repetition into active learning and it gives your brain a workout at the same time. Good luck!
