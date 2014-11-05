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
