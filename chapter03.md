## Building an authentication system
In the last chapter, we create a `UserProfile` model so we can store some extra information about our users. 

Now we will build an authentication system that let's users register, log in, and log out. We will cover write API views with Django REST Framework, creating an `Authentication` service with AngularJS and finally the templates and controllers that will display various forms to the client.

## Registration
Because we can't log in users that don't exist, it makes sense to start with registration. 

To register a user, we need an API endpoint that will create the user, an AngularJS service to make an AJAX request to the API and a registration form. Let's make the API endpoint first.

## Registration API Views and URLs
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

## Authentication Service
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

## Registration Controller and Template
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

## Registration Routes and Modules
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

## Hash routing
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

## Include new .js files
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

Congratulations! You're well on your way to having a fully working web application built with Django and AngularJS. But we aren't out of the woods just yet! Let's keep on truckin'.

{x: high_five_1}
Give yourself a high five. Seriously. Do it now.


