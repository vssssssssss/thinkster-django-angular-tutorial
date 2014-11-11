# User profiles
The only thing left to do at this point is make a profile page for your users. We will cover that now.

We need a few things to accomplish this. 

All of the models and serializers we will use already exist, so we are good on that front. However, there are two new views that we will need to make: one for retrieving and updating user profiles and one for deleting users. 

Other things we will need to create include a service for retrieving user profiles and a couple of controllers with their associated templates.

This is the final stretch. Let's jump in!

## User profile view
The first thing we will work on is a view for retrieving and updating a user's profile. 

Open up `authentication/views.py` and add the following view and imports:

    from authentication.models import UserProfile
    from authentication.permissions import IsAuthenticatedAndOwnsProfile
    from authentication.serializers import UserProfileSerializer

    class UserProfileRetrieveUpdateView(generics.RetrieveUpdateAPIView):
        lookup_field = 'user__username'
        queryset = UserProfile.objects.all()
        serializer_class = UserProfileSerializer

        def get_permissions(self):
            if self.request.method in permissions.SAFE_METHODS:
                return (permissions.AllowAny(),)
            return (IsAuthenticatedAndOwnsProfile(),)

Most of this will be familiar. Here are a few things to consider:

    lookup_field = 'user__username'

Normally we look up objects by their primary key. However, doing so results in URLs like `/api/v1/users/1`. URLs are much more readable if we use the user's username: `/api/v1/users/james`. To do these, we set the lookup field to `user__username`.

Note the double underscores in the lookup field. In Django this means "grab the associated user object and return it's username attribute."

    return (IsAuthenticatedAndOwnsProfile(),)

This is a custom permission similar to the one we wrote earlier. We will make this one in just a moment.

If the request method is not one of `permissions.SAFE_METHODS`, then the user must be trying to update their profile. In this case, we want to make sure they are authenticated and that this is their profile. 

## IsAuthenticatedAndOwnsProfile permission
Create `authentication/permissions.py` with the following content:

    from rest_framework import permissions


    class IsAuthenticatedAndOwnsProfile(permissions.BasePermission):
        def has_object_permission(self, request, view, obj):
            if not request.user and request.user.is_authenticated():
                return False
            return obj == request.user.profile

This permission is fairly straight forward. If the user is not authenticated, then there is no reason to check if they are trying to update a profile that is not theirs. They shouldn't be updating anything at all.

If the user is authenticated then we make sure this is their profile.

## User delete view
We also need a view for deleting users, in case someone wants to delete their account.

Open `authentication/views.py` and add the following view:

    class UserDestroyView(generics.DestroyAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

Since this is a simple view we will let Django REST Framework handle it for us.

## API endpoints
Now you need to add your two new views to your API.

Open up `thinkster_django_angular_boilerplate/urls.py` and add these new routes and imports:

    from authentication.views import UserDestroyView, UserProfileRetrieveUpdateView

    urlpatterns = patterns(
        # ...
        url(r'^api/v1/users/(?P<pk>[0-9]+)/$',
            UserDestroyView.as_view(), name='user-destroy'),
        url(r'^api/v1/users/(?P<user__username>[a-zA-Z0-9_@+-]+)$',
            UserProfileRetrieveUpdateView.as_view(), name='profile'),
    )

## Profiles module
We will be creating a service and a couple of controllers relating to user profiles, so let's go ahead and define the modules we will need.

Create `static/javascripts/profiles/profiles.module.js` with the following content:

    angular.module('borg.profiles', [
      'borg.profiles.controllers',
      'borg.profiles.services'
    ]);

    angular.module('borg.profiles.controllers', []);
    angular.module('borg.profiles.services', []);

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/profiles.module.js' %}"></script>

## Profile service
With the module definitions in place, we are ready to create the `Profile` service that will communicate with our API.

Create `static/javascripts/profiles/services/profile.service.js` with the following contents:

    angular.module('borg.profiles.services')
      .service('Profile', function ($http) {
        var Profile = {
          destroy: function (profile) {
            return $http.delete('/api/v1/users/' + profile.id + '/');
          },

          get: function (username) {
            return $http.get('/api/v1/accounts/' + username + '/');
          },

          update: function (profile) {
            return $http.put('/api/v1/accounts/' + profile.username + '/', profile);
          }
        };

        return Profile;
      });

We aren't doing anything special here. Each of these API calls is a basic CRUD operation, so we get away with not having much code.

Add this file to `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/services/profile.service.js' %}"></script>

## Profile controller
The next step is to create the controller that will use the service we just created, along with the `Thought` service, to retrieve the data we want to display.

Create `static/javascripts/profiles/controllers/profile.controller.js` with the following content:

    angular.module('borg.profiles.controllers')
      .controller('ProfileController', function ($location, $routeParams, $scope, Profile, Snackbar, Thoughts) {
        var username = $routeParams.username.substr(1);

        Profile.get(username).then(
          function (data, status, headers, config) {
            $scope.profile = data.data;
          },
          function (data, status, headers, config) {
            $location.url('/');
            Snackbar.error('That user does not exist.');
          }
        );

        Thoughts.get(username).then(
          function (data, status, headers, config) {
            $scope.thoughts = data.data;
          },
          function (data, status, headers, config) {
            Snackbar.error(data.error);
          }
        );
      });

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/controllers/profile.controller.js' %}"></script>


In this controller, we are sending two AJAX requests at the same time: one for the user's profile and one for their posts. <<-- REFACTOR THIS SHIT. TWO REQUESTS IS RIDICULOUS. DO YOU EVEN PREFETCH_RELATED BRO?

If `Profile.get()` gets an error response, we can reasonably assume that the user did not exist, so we redirect to the index page and show a snackbar telling the user what happened.

## Profile template
To accompany our controller, we will need a template.

Create `static/templates/profiles/profile.html` with the following content:

    <div ng-show="account">
      <div class="jumbotron profile__header">
        <h1 class="profile__username">{{ account.username }}</h1>
        <p class="profile__tagline">{{ account.tagline }}</p>
      </div>

      <thoughts thoughts="thoughts"></thoughts>
    </div>

This will render a header with the username and tagline of the profile owner, followed by a list of their posts. The posts are rendered using the directive we created earlier for the index page.

## Profile route
Open 'static/javascripts/borg.routes.js` and add the following route:

    .when('/+:username', {
      controller: 'ProfileController',
      templateUrl: '/static/templates/profiles/profile.html'
    })

## Go check out a profile
Open your browser and load up `http://localhost:8000`. Click on the `+<username>` link in the navigation bar to view your profile.
