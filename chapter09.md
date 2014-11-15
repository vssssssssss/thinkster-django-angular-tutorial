# Displaying user profiles
In addition to being able to make new posts, users should be able to view each others profiles. We need a few things to accomplish this. 

First, we need two new views: one for retrieving and updating profiles and one for deleting them. Then we will create a service for retrieving profiles and the controllers and templates for displaying them.

## Making a view to retrieving and updating profiles
Open up `authentication/views.py` and add the following view and imports:

    from rest_framework import generics, permissions

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

## Making the IsAuthenticatedAndOwnsProfile permission
Create `authentication/permissions.py` with the following content:

    from rest_framework import permissions


    class IsAuthenticatedAndOwnsProfile(permissions.BasePermission):
        def has_object_permission(self, request, view, obj):
            if request.user and request.user.is_authenticated():
                return obj == request.user.profile
            return False

This permission is fairly straight forward. If the user is not authenticated, then there is no reason to check if they are trying to update a profile that is not theirs. They shouldn't be updating anything at all.

If the user is authenticated then we make sure this is their profile.

## Making an API view to delete user profiles
We also need a view for deleting users, in case someone wants to delete their account.

Open `authentication/views.py` and add the following view:

    class UserDestroyView(generics.DestroyAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

Since this is a simple view we will let Django REST Framework handle it for us.

## Adding API endpoints for retrieving, updating and deleting user profiles
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

## Making the profile modules
We will be creating a service and a couple of controllers relating to user profiles, so let's go ahead and define the modules we will need.

Create `static/javascripts/profiles/profiles.module.js` with the following content:

    (function () {
      'use strict';

      angular
        .module('thinkster.profiles', [
          'thinkster.profiles.controllers',
          'thinkster.profiles.services'
        ]);

      angular
        .module('thinkster.profiles.controllers', []);

      angular
        .module('thinkster.profiles.services', []);
    })();

{x: profile_modules}
Define the modules needed for profiles in `profiles.module.js`

As always, don't forget to register `thinkster.profiles` as a dependency of `thinkster` in `thinkster.js`:

    angular
      .module('thinkster', [
        'thinkster.config',
        'thinkster.routes',
        'thinkster.authentication',
        'thinkster.layout',
        'thinkster.posts',
        'thinkster.profiles'
      ]);

{x: thinkster_profile_module}
Register `thinkster.profiles` as a dependency of the `thinkster` module

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/profiles.module.js' %}"></script>

## Making a Profile factory
With the module definitions in place, we are ready to create the `Profile` service that will communicate with our API.

Create `static/javascripts/profiles/services/profile.service.js` with the following contents:

    /**
    * Profile
    * @namespace thinkster.profiles.services
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.profiles.services')
        .factory('Profile', Profile);

      Profile.$inject = ['$http'];

      /**
      * @namespace Profile
      */
      function Profile($http) {
        /**
        * @name Profile
        * @desc The factory to be returned
        * @memberOf thinkster.profiles.services.Profile
        */
        var Profile = {
          destroy: destroy,
          get: get,
          update: update
        };

        return Profile;

        /////////////////////

        /**
        * @name destroy
        * @desc Destroys the given profile
        * @param {Object} profile The profile to be destroyed
        * @returns {Promise}
        * @memberOf thinkster.profiles.services.Profile
        */
        function destroy(profile) {
          return $http.delete('/api/v1/users/' + profile.id + '/');
        }


        /**
        * @name get
        * @desc Gets the profile for user with username `username`
        * @param {string} username The username of the user to fetch
        * @returns {Promise}
        * @memberOf thinkster.profiles.services.Profile
        */
        function get(username) {
          return $http.get('/api/v1/users/' + username + '/');
        }


        /**
        * @name update
        * @desc Update the given profile
        * @param {Object} profile The profile to be updated
        * @returns {Promise}
        * @memberOf thinkster.profiles.services.Profile
        */
        function update(profile) {
          return $http.put('/api/v1/users/' + profile.username + '/', profile);
        }
      }
    })();

{x: profiles_factory}
Create a new factory called `Profiles` in `static/javascripts/profiles/services/profiles.service.js`

We aren't doing anything special here. Each of these API calls is a basic CRUD operation, so we get away with not having much code.

Add this file to `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/services/profile.service.js' %}"></script>

{x: profiles_service_include_js}
Include `profiles.service.js` in `javascripts.html`

## Making an interface for user profiles
Create `static/templates/profiles/profile.html` with the following content:

    <div class="profile" ng-show="vm.profile">
      <div class="jumbotron profile__header">
        <h1 class="profile__username">+{{ vm.profile.username }}</h1>
        <p class="profile__tagline">{{ vm.profile.tagline }}</p>
      </div>

      <posts posts="posts"></posts>
    </div>

{x: profile_template}
Make a template for displaying profiles in `static/templates/profiles/profile.html`

This will render a header with the username and tagline of the profile owner, followed by a list of their posts. The posts are rendered using the directive we created earlier for the index page.

## Controlling the profile interface with ProfileController
The next step is to create the controller that will use the service we just created, along with the `Post` service, to retrieve the data we want to display.

Create `static/javascripts/profiles/controllers/profile.controller.js` with the following content:

    /**
    * ProfileController
    * @namespace thinkster.profiles.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.profiles.controllers')
        .controller('ProfileController', ProfileController);

      ProfileController.$inject = ['$location', '$routeParams', 'Posts', 'Profile', 'Snackbar'];

      /**
      * @namespace ProfileController
      */
      function ProfileController($location, $routeParams, Posts, Profile, Snackbar) {
        var vm = this;

        vm.profile = undefined;
        vm.posts = [];

        activate();

        /**
        * @name activate
        * @desc Actions to be performed when this controller is instantiated
        * @memberOf thinkster.profiles.controllers.ProfileController
        */
        function activate() {
          var username = $routeParams.username.substr(1);

          Profile.get(username).then(profileSuccessFn, profileErrorFn);
          Posts.get(username).then(postsSuccessFn, postsErrorFn);

          /**
          * @name profileSuccessProfile
          * @desc Update `profile` on viewmodel
          */
          function profileSuccessFn(data, status, headers, config) {
            vm.profile = data.data;
          }


          /**
          * @name profileErrorFn
          * @desc Redirect to index and show error Snackbar
          */
          function profileErrorFn(data, status, headers, config) {
            $location.url('/');
            Snackbar.error('That user does not exist.');
          }


          /**
            * @name postsSucessFn
            * @desc Update `posts` on viewmodel
            */
          function postsSuccessFn(data, status, headers, config) {
            vm.posts = data.data;
          }


          /**
            * @name postsErrorFn
            * @desc Show error snackbar
            */
          function postsErrorFn(data, status, headers, config) {
            Snackbar.error(data.data.error);
          }
        }
      }
    })();

{x: profile_controller}
Create a new controller called `ProfileController` in `static/javascripts/profiles/controllers/profile.controller.js`

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/controllers/profile.controller.js' %}"></script>

{x: profile_controller_include_js}
Include `profile.controller.js` in `javascripts.html`

## Making a route for viewing user profiles
Open `static/javascripts/thinkster.routes.js` and add the following route:

    .when('/+:username', {
      controller: 'ProfileController',
      controllerAs: 'vm',
      templateUrl: '/static/templates/profiles/profile.html'
    })

{x: profile_route}
Make a route for viewing user profiles

## Checkpoint
To view your profile, direct your browser to `http://localhost:8000/+<username>`. If the page renders, everything is good!

{x: checkpoint_display_user_profile}
Visit your profile page at `http://localhost:8000/+<username>`
