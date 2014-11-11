# Creating new posts
Cool. We can retrieve posts from the server and display them on the client, but what about creating them? We already have the endpoint and service taken care of, so let's finish things off by adding an interface to the client.

The first thing we need to do is add a button that the user will click to access the form. When the button is clicked, a modal will pop up prompting the user to enter the text for their new post with a button to submit the form.

## New Thought Button
To get started, open `static/templates/thoughts/new-thought.html` and add the following to the bottom of the file:

    <a class="btn btn-primary btn-fab btn-raised mdi-content-add btn-new-thought"
       href="javascript:void(0)"
       ng-show="isAuthenticated"
       ng-dialog="/static/templates/thoughts/new-thought.html"
       ng-dialog-controller="NewThoughtController"></a>

Earlier we talked about how the `ngDialog` directive would be used for creating new posts. `ngDialog` was included as part of the boilerplate and we won't go into detail on it's full API.

All you need to know is that, at minimum, we want to tell `ngDialog` which template and controller to use and that is what we do here.

Also note that we make use of `ng-show` to hide this button from non-authenticated users.

Now let's create the controller and template for this modal.

## New Thought Controller
Create `static/javascripts/thoughts/controller/new-thought.controller.js` with the following content:

    angular.module('borg.thoughts.controllers')
      .controller('NewThoughtController', function ($rootScope, $scope, Authentication, Snackbar, Thoughts) {
        $scope.submit = function () {
          $rootScope.$broadcast('thought.created', {
            content: $scope.content,
            author: {
              username: Authentication.getAuthenticatedUser().username
            }
          });

          $scope.closeThisDialog();

          Thoughts.create($scope.content).then(
            function (data, status, headers, config) {
              Snackbar.show('Success! Your thought has been uploaded to the Collective.');
            },
            function (data, status, headers, config) {
              $rootScope.$broadcast('thought.created.error');
              Snackbar.error(data.error);
            }
          );
        };
      });

There are a few things going on here that we should talk about.

    $rootScope.$broadcast('thought.created', {
      content: $scope.content,
      author: {
        username: Authentication.getAuthenticatedUser().username
      }
    });

Earlier we set up an event listener in `IndexController` that listened for the `thought.created` event and then pushed the new thought onto the front of `$scope.thoughts`. Let's look at this a little more closely, as this turns out to be an important feature of rich web applications.

What we are doing here is being *optimistic* that the API response from `Thoughts.create()` will contain a 200 status code telling us everything went according to plan. This may seem like a bad idea at first. Something could go wrong during the request and then our data is stale. Why don't we just wait for the response?

When I said we are increasing the *perceived* performance of our app, this is what I was talking about. We want the user to *perceive* the response as instant.

The fact of the matter is that this call will rarely fail. There are really only two cases where this will reasonably fail: either the user is not authenticated or the server is down.

In the case where the user is not authenticated, they shouldn't be submitting new posts anyways. Consider the error to be a small punishment for the user doing things they shouldn't.

If the server is down, then there is nothing we can do. Unless the user already had the page loaded before the server crashed, they wouldn't be able to see this page anyways.

Other things that could possibly go wrong make up such a small percentage that we are willing to allow a slightly worse experience to make the experience better for the 99.9% of cases where everything is working properly.

Furthermore, the object we pass as the second argument is meant to emulate the response from the server. This is not the best design pattern because it assumes we know what the response will look like. If the response changes, we have to update this code. However, given what we have, this is an acceptable cost.

So what happens when the API call returns an error?

    $rootScope.$broadcast('thought.created.error');

If the error callback is triggered, then we will broadcast a new event: `thought.created.error`. The event listener we set up earlier will be trigger by this event and remove the post at the front of `$scope.thoughts`. We will also show the error message to the user to let them know what happened.

    $scope.closeThisDialog();

This is a method provided by `ngDialog`. All it does is close the model we have open.

## New Post Template
With the controller in place, let's add a template for the modal.

Create `static/templates/thoughts/new-thought.html` with the following contents:

    <form role="form" class="thoughts__add-new" ng-submit="submit()">
      <div class="form-group">
        <label for="thought__content">Thought</label>
        <textarea class="form-control" id="thought__content" rows="3" placeholder="We are borg ..." ng-model="content"></textarea>
      </div>

      <div class="form-group">
        <button type="submit" class="btn btn-primary">
          Submit
        </button>
      </div>
    </form>

We now have every thing we need to create new posts. Go ahead and try it out by running your Django server and loading up `http://localhost:8000/` in your browser.

You'll see the button we added earlier on the left side of the screen (scroll down if you have a number of posts already). Just click it and fill out the form.
