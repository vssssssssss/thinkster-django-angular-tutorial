# Rendering the Borg's Thoughts
Until now, the index page has been empty. Now that we have handled authentication and the backend details for the `Thought` model, it's time to give our users something to interact with. We will do this by creating a service that handles retrieving and creating `Thought`s and some controllers and directives for handling how the data is displayed.

## Thoughts module
Let's define the thoughts modules.

Create a file in `static/javascripts/thoughts` called `thoughts.module.js` and add the following:

    angular.module('borg.thoughts', [
      'borg.thoughts.controllers',
      'borg.thoughts.directives',
      'borg.thoughts.services'
    ]);

    angular.module('borg.thoughts.controllers', []);
    angular.module('borg.thoughts.directives', ['ngDialog']);
    angular.module('borg.thoughts.services', []);

{x: thoughts_module}
Define the `borg.thoughts` module

Remember to add `borg.thoughts` as a dependency of `borg` in `borg.js`:

    angular.module('borg', [
      // ...
      'borg.thoughts'
    ]);

{x: thoughts_module_dep_borg}
Add `borg.thoughts` as a dependency of the `borg` module

There are two things worth noting about this module.

First, we have created a module named `borg.thoughts.directives`. As you probably guessed, this means we will introduce the concept of directives to our app in this chapter.

Secondly, the `borg.thoughts.directives` module requires the `ngDialog` module. `ngDialog` is included in the boilerplate project and handles the display of modals. We will use a modal in the next chapter when we write the code for creating new posts.

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/thoughts/thoughts.module.js' %}"></script>

{x: thoughts_include_module}
Include `thoughts.module.js` in `javascripts.html`

## Thoughts Service
Before we can render anything, we need to transport data from the server to the client. As mentioned, Angular's services are how we accomplish this.

Create a file at `static/javascripts/thoughts/services` called `thoughts.service.js` and add the following:

    angular.module('borg.thoughts.services')
      .service('Thoughts', function ($http) {
        var Thoughts = {
          all: function () {
            return $http.get('/api/v1/thoughts/');
          },

          create: function (content) {
            return $http.post('/api/v1/thoughts/', {
              content: content
            });
          }
        };

        return Thoughts;
      });

{x: thoughts_service}
Create the `Thoughts` service

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/thoughts/services/thoughts.service.js' %}"></script>

{x: thoughts_service_include_javascripts}
Include `thoughts.service.js` in `javascripts.html`

This code should look pretty familiar. It is very similar to the services we created before.

The `Thoughts` service only has two methods for now: `all` and `create`.

On the index page, we will use `Thoughts.all()` to get the list of objects we want to display. We will also use `Thoughts.create()` to let users add their own thoughts.

## IndexController
Now that we can transport `Thought` data from the server to the client, let's start adding some content to the index page. This will involve creating an `IndexController` to handle fetching data when the page is loaded. After that we will create some directives for displaying the data properly.

Create a file in `static/javascripts/static/controllers/` called `index.controller.js` and add the following:

    angular.module('borg.static.controllers')
      .controller('IndexController', function ($scope, Authentication, Snackbar, Thoughts) {
        $scope.isAuthenticated = !!Authentication.getAuthenticatedUser();

        Thoughts.all().then(
          function (data, status, headers, config) {
            $scope.thoughts = data.data;
          },
          function (data, status, headers, config) {
            Snackbar.snackbar('ERROR: ' + data.error, {
              timeout: 3000
            });
          }
        );

        $scope.$on('thought.created', function (e, thought) {
          $scope.thoughts.unshift(thought);
          $scope.thoughts = $scope.thoughts.slice(0);
        });

        $scope.$on('thought.created.error', function () {
          $scope.thoughts.shift();
          $scope.thoughts = $scope.thoughts.slice(0);
        });
      });

{x: index_controller}
Create the `IndexController` controller

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/static/controllers/index.controller.js' %}"></script>

{x: index_controller_include_javascripts}
Include `index.controller.js` in `javascripts.html`

Let's touch on a few things here.

    $scope.isAuthenticated = !!Authentication.getAuthenticatedUser();

In one of the templates we will create soon, we will add a button that lets the user create a new post. We should only allow a user to create a post if they are authenticated. Because all we need to know about the user is whether they are authenticated, we don't actually need the result of `Authentication.getAuthenticatedUser()`. 

If you remember, `getAuthenticatedUser` will return the user if they are authenticated and `undefined` if they aren't. Using `!!` turns any value into a boolean (true or false) using the values "truthiness". For example, `undefined` is "false-y" in JavaScript. Because of this, `!undefined` is `true`, but this is the negative value of `undefined` -- the opposite of what we want. Negating this value again we get the positive value, `!!undefined` or `false`. This is what we will use in our template to decide whether we are going to show the button to create a new post.

    $scope.$on('thought.created', function (e, thought) {
      $scope.thoughts.unshift(thought);
      $scope.thoughts = $scope.thoughts.slice(0);
    });

Later, when we get around to creating a new post, we will fire off an event called `post.created` when the user creates a post. By catching this event here, we can add this new thought to the front of the `$scope.thoughts` array. This will prevent us from having to make an extra API request to the server for updated data. We will talk about this more shortly, but for now you should know that we do this to increase the *perceived* performance of our application.

As you will see in a few minutes, we are going to watch `$scope.thoughts` for changes. Because of the way `$scope.$watch` works in Angular, we need to make `$scope.thoughts` point to a copy of itself to avoid the cost of deep-watching. This is an expensive, albeit acceptable trade off.

    $scope.$on('thought.created.error', function () {
      $scope.thoughts.shift();
      $scope.thoughts = $scope.thoughts.slice(0);
    });

Analogous to the previous event listener, this one will remove the post at the front of `$scope.thoughts` if the API request returns an error status code.

## Index template
The template for the index page is just about as short as you could want. We will go into more detail in a moment we create our first directive.

Create `static/templates/static/index.html` with the following contents:

    <thoughts thoughts="thoughts"></thoughts>

{x: index_template}
Create the index template

Yep. That's all. I told you there wasn't much to it. We will add a little more later, but not much. Most of what we need will be in the template we create for the thoughts directive next.

## Posts directive
Directives are widely considered to be one of the more difficult concepts in AngularJS. Personally, I attribute this to what I consider a sub-optimal API. You'll see what I mean in a moment.

Create `static/javascripts/thoughts/directives/thoughts.directive.js` with the following contents:

    angular.module('borg.thoughts.directives')
      .directive('thoughts', function () {
        return {
          controller: 'ThoughtsController',
          scope: {
            thoughts: '='
          },
          restrict: 'E',
          templateUrl: '/static/templates/thoughts/thoughts.html' 
        };
      });

{x: thoughts_directive}
Create a `thoughts` directive

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/thoughts/directives/thoughts.directive.js' %}"></script>

{x: thoughts_directive_include_js}
Include `thoughts.directive.js` in `javascripts.html`

There are two parts of the directives API that I want to touch on: `scope` and `restrict`.

    scope: {
      thoughts: '='
    },

`scope` defines the scope of this directive, similar to how `$scope` works for controllers. The difference is that, in a controller, a new scope is implicitly created. For a directive, we have the option of explicitly defining our scopes and that's what we do here.

The second line, `thoughts: '='` simply means that we want to set `$scope.thoughts` to the value passed in through the `thoughts` attribute in the template that we made earlier.

    restrict: 'E',

`restrict` tells Angular how we are allowed to use this directive. In our case, we set the value of `restrict` to `E` (for element) which means Angular should only match the name of our directive with the name of an element: `<thoughts></thoughts>`. 

Another common option is `A` (for attribute), which tells Angular to only match the name of the directive with the name of an attribute. `ngDialog` uses this option, as we will see shortly.

## PostsController
The directive we just created requires a controller called `PostsController`. 

Create `static/javascripts/thoughts/controllers/posts.controller.js` with the following content:

    angular.module('borg.thoughts.controllers')
      .controller('ThoughtsController', function ($scope) {
       $scope.nodes = [];

        var calculateNumberOfColumns = function () {
          var width = $(window).width();

          if (width >= 1200) { return 4; } 
          else if (width >= 992) { return 3; } 
          else if (width >= 768) { return 2; } 
      
          return 1;
        };

        var getSmallestNode = function () {
          var scores = $scope.nodes.map(function (node) {
            var sum = function (a, b) { return a + b; };

            var lengths = node.map(function (element) {
              return element.content.length;
            });

            return lengths.reduce(sum, 0) * node.length;
          });

          return scores.indexOf(
            Math.min.apply(this, scores)
          );
        };

        var render = function (newValue, oldValue) {
          if (newValue !== oldValue) {
            var thoughts = newValue;

            $scope.nodes = [];

            var numberOfColumns = calculateNumberOfColumns();

            for (var i = 0; i < numberOfColumns; ++i) {
              $scope.nodes.push([]);
            }

            for (var i = 0; i < thoughts.length; ++i) {
              var node = getSmallestNode();

              $scope.nodes[node].push(thoughts[i]);
            }
          }
        };


        $scope.$watch(function () { return $scope.thoughts; }, render);
        $scope.$watch(function () { return $(window).width(); }, render);
      });

{x: posts_controller}
Create a `PostsController` controller

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/thoughts/controllers/thoughts.controller.js' %}"></script>

{x: posts_controller_include_js}
Include `posts.controller.js` in `javascripts.html`

It isn't worth taking the time to step through this controller line-by-line. Suffice it to say that this controller presents an algorithm for ensuring the columns of posts are of approximately equal height.

## Posts template
In our directive we defined a `templateUrl` that doesn't match any of our existing templates. Let's go ahead and make a new one.

Create `static/templates/thoughts/thoughts.html` with the following content:

    <div class="row" ng-cloak>
      <div ng-repeat="node in nodes" ng-show="thoughts && thoughts.length">
        <div class="col-xs-12 col-sm-6 col-md-4 col-lg-3">
          <div ng-repeat="thought in node">
            <thought thought="thought"></thought>
          </div>
        </div>
      </div>

      <div ng-hide="thoughts && thoughts.length">
        <div class="col-sm-12" style="text-align: center;">
          <em>The Borg has no thoughts.</em>
        </div>
      </div>
    </div>

{x: posts_template}
Create a template for the `posts` directive

A few things worth noting:

1. We use the `ng-cloak` directive to prevent flashing since this directive will be used on the first page loaded.
2. We will need to create a `thought` directive for rendering each individual post.
3. If no thoughts are present, we render a message informing the user.

## Post directive
In the template for the posts directive, we use another directive called `post`. Let's create that.

Create `static/javascripts/thoughts/directives/thought.directive.js` with the following content:

    angular.module('borg.thoughts.directives')
      .directive('thought', function () {
        return {
          scope: {
            thought: '='
          },
          restrict: 'E',
          templateUrl: '/static/templates/thoughts/thought.html'
        };
      });

{x: post_directive}
Create a `post` directive

Include this file in `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/thoughts/directives/thought.directive.js' %}"></script>

{x: post_directive_include_js}
Include `post.directive.js` in `javascripts.html`

There is nothing new worth discussing here. This directive is almost identical to the previous one. The only difference is we use a different template.

## Post template
Like we did for the `posts` directive, we now need to make a template for the `post` directive.

Create `static/templates/thoughts/thought.html` with the following content:

    <div class="row">
      <div class="col-sm-12">
        <div class="well thought">
          <div class="thought__meta">
            <a href="/+{{ thought.author.username }}">
              +{{ thought.author.username }}
            </a>
          </div>

          <div class="thoguht__content">
            {{ thought.content }}
          </div>
        </div>
      </div>
    </div>

{x: post_template}
Create a template for the `post` directive

There are a few special CSS classes here that we will address later, but that's about it.

## Index route
Now that we have all of our services and directives ready to go, let's specify the controller and template for the the index route.

Open `static/javascripts/borg.routes.js` and add the following route:

    .when('/', {
      controller: 'IndexController',
      templateUrl: '/static/templates/static/index.html'
    })

{x: index_route}
Add a route to `borg.routes.js` for the `/` path
