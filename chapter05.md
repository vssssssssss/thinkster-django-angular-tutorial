# Modeling the Borg's Thoughts
In this chapter we will make a new app and create a `Thought` model similar to a status on Facebook or a tweet on Twitter. After we create our model we will move on to serializing `Thought`s and then we will create a few new endpoints for our API.

Why waste time? Let's jump right in.

## App
First things first: go ahead and create a new app called `thoughts`.

    $ python manage.py start app thoughts

{x: django_app_thoughts}
Create a new app named `thoughts`

Remember: whenever you create a new app you have to add it to the `INSTALLED_APPS` setting. Open `thinkster_django_angular_boilerplate/settings.py` and modify it like so:

    INSTALLED_APPS = (
        # ...
        'thoughts',
    )

## What does the Thought model look like?
When you created the `thoughts` app Django made a new file called `thoughts/models.py`. Go ahead and open it up and add the following:

    from django.db import models
    from django.db.models.signals import pre_delete
    from django.dispatch import receiver

    from authentication.models import UserProfile


    class Thought(models.Model):
        author = models.ForeignKey(UserProfile)
        content = models.TextField()

        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)

        def __unicode__(self):
            return '{0}'.format(self.content)

        @receiver(pre_delete, sender=UserProfile)
        def delete_thoughts_for_account(sender, instance=None, **kwargs):
            if instance:
                thoughts = Thought.objects.filter(author=instance)
                thoughts.delete()

{x: django_model_thought}
Create a `Thought` model

Our method of walking through the code line-by-line is working well so far. Why mess with a good thing? Let's do it.

    author = models.ForeignKey(UserProfile)

When we created the `UserProfile` model we associated each `UserProfile` with a `User`. This is called a one-to-one relationship. Because each user can have a number of `Thought`s, we want to set up a different kind of relationship: a many-to-one.

The way to do this in Django is with using a `ForeignKey` field to associate each `Thought` with a `UserProfile`. 

Django is smart enough to know the foreign key we've set up here should be reversible. That is to say, given a `UserProfile`, you should be able to access that user's `Thought`s. In Django these `Thought` objects can be accessed through `UserProfile.thought_set` (not `UserProfile.thoughts`).

    @receiver(pre_delete, sender=UserProfile)
    def delete_thoughts_for_profile(sender, instance=None, **kwargs):
        if instance:
            thoughts = Thought.objects.filter(author=instance)
            thoughts.delete()

This `pre_delete` hook is similar to the one we set up for `UserProfile`. The difference this time is that we delete the `Thought` objects associated with a `UserProfile` before the `UserProfile` is deleted.

It's interesting to note that this creates a sort of chain. Before a `User` is deleted, the associated `UserProfile` is deleted. Before a `UserProfile` is deleted, the associated `Thought` objects are deleted.

Now that the model exists, don't forget to migrate.

    $ python manage.py makemigrations
    $ python manage.py migrate

{x: django_model_thought_migrate}
Create migrations for `Thought` and apply them

## Serializing the Thought model
Create a new file in `thoughts/` called `serializers.py` and add the following:

    from rest_framework import serializers

    from authentication.serializers import UserProfileSerializer
    from thoughts.models import Thought


    class ThoughtSerializer(serializers.ModelSerializer):
        author = UserProfileSerializer(required=False)

        class Meta:
            model = Thought

            fields = ('id', 'author', 'content', 'created_at', 'updated_at')
            read_only_fields = ('id', 'author', 'created_at', 'updated_at')

        def get_validation_exclusions(self, *args, **kwargs):
            exclusions = super(ThoughtSerializer, self).get_validation_exclusions()

            return exclusions + ['author']

{x: django_serializer_thoughtserializer}
Create a serializer called `ThoughtSerializer`

There isn't much here that's new, but there is one line in particular I want to look at.

    author = UserProfileSerializer(required=False)

We explicitly defined a number of fields in our `UserProfileSerializer` from before, but this definition is a little different.

When serializing a `Thought` object, we want to include all of the author's information. Within Django REST Framework, this is known as a nested relationship. Basically, we are serializing the `UserProfile` related to this `Thought` and including it in our JSON.

We pass `required=False` here because we will set the author of this post automatically.

     def get_validation_exclusions(self, *args, **kwargs):
          exclusions = super(ThoughtSerializer, self).get_validation_exclusions()

          return exclusions + ['author']

For the same reason we use `required=False`, we must also add `author` to the list of validations we wish to skip.

At this point, feel free to open up your shell with `python manage.py shell` and play around with creating and serializing `Thought` objects.

    >>> from authentication.models import UserProfile
    >>> from thoughts.models import Thought
    >>> from thoughts.serializers import ThoughtSerializer
    >>> u = UserProfile.objects.get(pk=1)
    >>> t = Thought.objects.create(author=u, content='You will be assimilated!')
    >>> s = ThoughtSerializer(t)
    >>> s.data

{x: django_shell_thought}
Play around with the `Thought` model and `ThoughtSerializer` serializer in Django's shell

## API Views
The next step in creating `Thought` objects is adding an API endpoint that will handle performing actions on the `Thought` model such as create or update.

Replace the contents of `thoughts/views.py` with the following:

    from rest_framework import generics, permissions

    from authentication.models import UserProfile
    from thoughts.models import Thought
    from thoughts.permissions import IsAuthenticatedAndOwnsObject
    from thoughts.serializers import ThoughtSerializer


    class ThoughtListCreateView(generics.ListCreateAPIView):
        queryset = Thought.objects.order_by('-created_at')
        serializer_class = ThoughtSerializer

        def get_permissions(self):
            if self.request.method == 'POST':
                return (permissions.IsAuthenticated(),)
            return (permissions.AllowAny(),)

        def pre_save(self, obj):
            obj.author = UserProfile.objects.get(user=self.request.user)
            return super(ThoughtListCreateView, self).pre_save(obj)


    class ThoughtRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):
        queryset = Thought.objects.all()
        serializer_class = ThoughtSerializer

        def get_permissions(self):
            if self.request.method not in permissions.SAFE_METHODS:
                return (IsAuthenticatedAndOwnsObject(),)
            return (permissions.AllowAny(),)

{x: django_view_thought_listcreate}
Create a view for listing and creating `Thought` objects

{x: django_view_thought_retrieveupdatedestroy}
Create a view for reading, updating and destroying `Thought` objects


Do these views look similar? They aren't that different than the ones we made to create `User` objects.

    def pre_save(self, obj):
        obj.author = UserProfile.objects.get(user=self.request.user)
        return super(ThoughtListCreateView, self).pre_save(obj)

When a `Thought` object is created it has to be associated with an author. Making the author type in their own username or id when creating adding a thought to the site would be a bad experience, so we handle this association for them.

`pre_save` is a method provided by the mixins used in `generics.ListCreateAPIView` (which we inherit from) that can be overridden. 

We take advantage of this feature to look up the currently logged in user's `UserProfile` and then use that profile as the `author` attribute for the `Thought` we are creating.

    def get_permissions(self):
        if self.request.method == 'POST':
            return (permissions.IsAuthenticated(),)
        return (permissions.AllowAny(),)

Only authenticated users should be allowed to create new `Thought`s. To satisfy this constraint we override `get_permissions`. 

If the request is a `POST` (the user is trying to create a new `Thought`), then we only allow authenticated users to continue. However, if the request is a `GET` (the only other HTTP verb this view accepts), then we will allow both authenticated and non-authenticated users to continue.

    class ThoughtRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):

That's quite a long name, huh?

Even though we create this view and add an endpoint for it, we will not be using it in our application. It is considered good practice to make sure the software interface you implement are complete. For us, that makes being able to create, read, update, destroy, and list `Thought` objects. 

We include this class to ensure we are providing a complete interface for handling `Thought` objects.

    def get_permissions(self):
        if self.request.method not in permissions.SAFE_METHODS:
            return (IsAuthenticatedAndOwnsObject(),)
        return (permissions.AllowAny(),)

Similar to the other view, we need to handle the permissions for updating and destroying `Thought`s here.

Django REST Framework provides a list of "safe" HTTP methods -- actions that do not modify data on the server, like `GET`. If the method for this request is not in that list, then we need to satisfy two constraints: the user must be authenticated and they must be own the object they are trying to modify.

`IsAuthenticatedAndOwnsObject` is a permission that we will implement ourselves now.

## Permissions
Create `permissions.py` in the `thoughts/` directory with the following content:

    from rest_framework import permissions

    from thoughts.models import Thought


    class IsAuthenticatedAndOwnsObject(permissions.BasePermission):
        def has_permission(self, request, view):
            if not request.user.is_authenticated():
                return False

            _id = self.kwargs['pk']

            return Thought.objects.filter(id=_id, author_id=request.user).exists()

{x: django_permission_isauthenticatedandownsobject}
Create an `IsAuthenticatedAndOwnsObject` permission

Let's step thought the code.

    class IsAuthenticatedAndOwnsObject(permissions.BasePermission):

When creating custom permissions with Django REST Framework, you should inherit from `permissions.BasePermission`. Django REST Framework, once again, handles a lot of boilerplate for us.

Custom permissions should return `True` if the user has permission to continue and `False` otherwise.

    if not request.user.is_authenticated():
        return False

Remember our first constraint? The user must be authenticated. `is_authenticated()` is a method provided by Django for handling this check.

    _id = self.kwargs['pk']

Presumably the request includes the `id` of the `Thought` we are modifying. We grab that now to make the next line shorter.

    return Thought.objects.filter(id=_id, author_id=request.user).exists()

This line satisfies our second constraint: the user must own this object. 

What we're doing here is filtering all `Thoughts` for anything matching the id and user provided. The `exists()` method returns `True` if our query found any matches and `False` otherwise.

## API Endpoints
With the views created, it's time to add the endpoints to our API.

Open `thinkster_django_angular_boilerplate/urls.py` and add the following urls:

    from thoughts.views import ThoughtListCreateView, \
        ThoughtRetrieveUpdateDestroyView

    urlpatterns = patterns(
        # ...,

        url(r'^api/v1/thoughts/$',
            ThoughtListCreateView.as_view(), name='thoughts'),
        url(r'^api/v1/thoughts/(?P<pk>[0-9]+)/$',
            ThoughtRetrieveUpdateDestroyView.as_view(), name='thought'),

        # ...,
    )

{x: django_url_thought_listcreate}
Create API endpoint for `ThoughtListCreateView`

{x: django_url_thought_retrieveupdatedestroy}
Create API endpoint for `ThoughtRetrieveUpdateDestroyView`
